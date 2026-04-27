# Adaptive Pricing Mechanism (APM)

An adaptive LP fee control system for **Uniswap v4 dynamic-fee pools**, combining an off-chain controller with an on-chain hook.

## Abstract

Adaptive Pricing Mechanism (APM) adjusts LP fees in response to short-horizon market conditions (volatility, flow imbalance, liquidity regime, and stale-price risk). The design separates:

- off-chain estimation and policy computation
- on-chain deterministic enforcement via a v4 hook

This document is aligned to Uniswap v4 core behavior and interface constraints.

## 1. Problem Statement

In concentrated-liquidity AMMs, LP outcomes are driven by fee income minus risk costs.

At a high level:

$$
\text{LP Net} \approx \int_0^T \phi_t V_t\,dt - \text{Inventory Risk} - \text{Adverse Selection (LVR-like losses)}
$$

Static fees cannot adapt to changing microstructure. During volatile or toxic-flow periods, static fees can underprice risk; during calm periods, static fees can overprice execution and lose flow.

APM targets state-dependent pricing while keeping on-chain execution simple and bounded.

## 2. Protocol Facts and Hard Constraints (Uniswap v4)

These are mandatory constraints for implementation correctness:

1. Dynamic-fee capability is chosen at pool creation and is immutable.
2. `beforeSwap` can override LP fee **per swap** only for dynamic-fee pools.
3. `beforeSwap` signature must follow v4 (`returns (bytes4, BeforeSwapDelta, uint24)`).
4. Fee override is only applied when:
   - pool is dynamic-fee,
   - override flag bit (`0x400000`) is set on returned `uint24`,
   - resulting fee is valid (`<= 1_000_000`).
5. LP fee units are **hundredths of a bip** (`1e-6`), max `1_000_000` (100%).
6. In hooks, `msg.sender` is `PoolManager`; original caller is passed as the `sender` argument.
7. Hook permissions are encoded in the deployed hook address bits.
8. One hook contract may serve multiple pools, so fee state must be keyed per pool.
9. `PoolManager.updateDynamicLPFee` can only be called by the hook for that pool.

## 3. System Architecture

APM is a closed-loop controller:

- **Plant**: Uniswap v4 pool (`PoolManager` + pool state)
- **Actuator**: Hook contract implementing `beforeSwap`
- **Controller**: Off-chain agent computing target fee
- **State estimator**: Off-chain feature pipeline from on-chain data

### 3.1 Control Objective

For each pool `i`, compute fee `\phi_{i,t}` that maximizes LP net outcome under execution constraints:

$$
\max_{\pi_\theta} \mathbb{E}[\text{FeeRevenue} - \text{RiskCosts} - \text{FlowLossPenalty}]
$$

Subject to:

- `\phi_{\min} \le \phi_{i,t} \le \phi_{\max}`
- bounded step size `|\phi_{i,t+1} - \phi_{i,t}| \le \Delta_{\max}`
- cooldown/update-rate limits

## 4. State Estimation and Policy

### 4.1 Features

For each pool over rolling windows:

- realized volatility `\sigma_t`
- swap arrival intensity `\lambda_t`
- in-range liquidity `L_t`
- flow imbalance proxy `\rho_t`
- external-vs-pool price divergence proxy `d_t`

### 4.2 Policy Form

Example parametric policy:

$$
\phi_t^{\text{raw}} = \phi_0 + \alpha \sigma_t^{\beta} + \gamma \rho_t + \kappa d_t
$$

Apply constraints:

$$
\phi_t^{\text{target}} = \text{clip}(\phi_t^{\text{raw}}, \phi_{\min}, \phi_{\max})
$$

$$
\phi_{t+1} = \text{clip}\Big(\phi_t + \text{clip}(\phi_t^{\text{target}}-\phi_t, -\Delta_{\max}, \Delta_{\max}),\; \phi_{\min},\; \phi_{\max}\Big)
$$

All fees are represented in v4 fee units (`uint24`, hundredths of a bip).

## 5. On-Chain Contract Design

### 5.1 Per-Pool State

```solidity
struct FeeState {
    uint24 currentFee;      // v4 units (1e-6)
    uint24 minFee;
    uint24 maxFee;
    uint24 maxStep;
    uint48 lastUpdate;
    uint32 cooldownSecs;
    bool enabled;
}

mapping(PoolId => FeeState) public feeState;
mapping(address => bool) public authorizedUpdaters;
```

Design requirement: state is keyed by `PoolId` to avoid cross-pool interference.

### 5.2 Controller Update Entry Point

```solidity
function updatePoolFee(PoolKey calldata key, uint24 newFee) external onlyAuthorized {
    PoolId id = key.toId();
    FeeState storage s = feeState[id];

    require(s.enabled, "pool disabled");
    require(newFee >= s.minFee && newFee <= s.maxFee, "bounds");
    require(block.timestamp >= s.lastUpdate + s.cooldownSecs, "cooldown");

    uint24 prev = s.currentFee;
    uint24 diff = prev > newFee ? prev - newFee : newFee - prev;
    require(diff <= s.maxStep, "step too large");

    s.currentFee = newFee;
    s.lastUpdate = uint48(block.timestamp);
}
```

### 5.3 Accurate `beforeSwap` Interface

```solidity
function beforeSwap(
    address sender,
    PoolKey calldata key,
    IPoolManager.SwapParams calldata params,
    bytes calldata hookData
) external override returns (bytes4, BeforeSwapDelta, uint24 lpFeeOverride) {
    require(msg.sender == address(poolManager), "only pool manager");

    PoolId id = key.toId();
    FeeState memory s = feeState[id];
    uint24 fee = s.enabled ? s.currentFee : s.minFee;

    // Per-swap override flag required by v4
    lpFeeOverride = fee | LPFeeLibrary.OVERRIDE_FEE_FLAG;

    return (this.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);
}
```

Notes:

- Returning a raw `uint24` without the override flag does not force per-swap fee override.
- This path must stay O(1), deterministic, and without heavy computation.

### 5.4 Optional Baseline Sync to PoolManager

APM may optionally persist fee as pool dynamic LP fee baseline:

```solidity
function syncDynamicFee(PoolKey calldata key) external onlyAuthorized {
    PoolId id = key.toId();
    poolManager.updateDynamicLPFee(key, feeState[id].currentFee);
}
```

This works only when `key.hooks == address(this)` and the pool was created as dynamic-fee.

## 6. Off-Chain Controller

Responsibilities:

1. ingest pool/tick/swap data via RPC or indexed data
2. compute state features and policy output
3. apply risk guards (bounds, step caps, cooldown awareness)
4. submit signed tx to hook (`updatePoolFee`)
5. monitor inclusion/finality and rollback handling

### 6.1 Update Loop (Pseudo-code)

```python
while True:
    state = fetch_pool_state(pool_key)
    features = compute_features(state)
    fee_target = policy(features, theta)
    fee_safe = apply_guards(fee_target)

    submit_tx(hook.updatePoolFee(pool_key, fee_safe))

    observe_outcomes()
    theta = update_params(theta)

    sleep(interval)
```

## 7. Security Model

### 7.1 Access Control

- controller keys are isolated from admin keys
- `authorizedUpdaters` can be hot-swapped
- admin actions behind multisig + timelock (recommended)

### 7.2 Manipulation and Data Integrity

- windowed estimators reduce single-block sensitivity
- cap fee jumps (`maxStep`) and update frequency (`cooldownSecs`)
- reject stale inputs and enforce max data age
- use robust price references for divergence features

### 7.3 Hook and Routing Semantics

- do not rely on `msg.sender` for trader identity
- `sender` is usually a router/aggregator, not necessarily end user
- if per-user logic is needed, require trusted router formats in `hookData`

### 7.4 Failure Modes

- controller offline: last valid on-chain fee remains active
- stuck high fee risk: include emergency clamp/fallback path
- key compromise: revoke updater + freeze pool config

## 8. Gas and Performance

- `beforeSwap`: constant-time read + bitwise OR
- no loops, no external calls, no expensive math in swap path
- write-heavy operations move to periodic controller updates

## 9. Product and Integration Risks

Technical correctness alone is insufficient for adoption.

Key risks:

1. Router/aggregator support for hooked pools may be uneven.
2. Competing dynamic-fee hooks already exist.
3. Without routing/distribution strategy, better pricing logic may not translate to sustained volume.

APM should include an explicit integration plan with routers/solvers and clear LP-facing performance reporting.

## 10. MVP Scope

Phase 1 (safe baseline):

- single pool
- bounded rule-based policy
- per-pool fee mapping
- monitoring + alerting

Phase 2 (adaptive):

- online parameter updates
- richer toxicity proxies
- multi-pool orchestration

Phase 3 (production hardening):

- formal verification targets for hook invariants
- external audit
- chaos and reorg simulation

## 11. Non-Goals

- on-chain ML or heavy real-time optimization in swap path
- replacing router infrastructure
- guaranteeing LP profitability under all market regimes

## 12. References

- Uniswap v4 Dynamic Fees: https://developers.uniswap.org/docs/protocols/v4/concepts/dynamic-fees
- Uniswap v4 Hooks Concepts: https://developers.uniswap.org/docs/protocols/v4/concepts/hooks
- Uniswap v4 Swap Hooks Guide: https://developers.uniswap.org/docs/protocols/v4/guides/hooks/swap-hooks
- Accessing `msg.sender` in v4 Hooks: https://developers.uniswap.org/docs/protocols/v4/guides/hooks/accessing-msg.sender
- Uniswap v4 Deployments: https://developers.uniswap.org/docs/protocols/v4/deployments
- v4 Core `IHooks.sol`: https://github.com/Uniswap/v4-core/blob/main/src/interfaces/IHooks.sol
- v4 Core `LPFeeLibrary.sol`: https://github.com/Uniswap/v4-core/blob/main/src/libraries/LPFeeLibrary.sol
- v4 Core `PoolManager.sol`: https://github.com/Uniswap/v4-core/blob/main/src/PoolManager.sol
- LVR paper (background): https://arxiv.org/abs/2208.06046
