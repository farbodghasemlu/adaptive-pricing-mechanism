# Adaptive Pricing Mechanism (APM)

Adaptive Pricing Mechanism (APM) is an LP fee control layer for Uniswap v4 dynamic-fee pools. It combines off-chain state estimation with on-chain enforcement through a hook, so fees can respond to market conditions without adding heavy logic to the swap path.

## Abstract

APM updates liquidity provider fees as market conditions change. The central idea is simple: when short-horizon risk rises, fees should rise; when risk normalizes, fees should relax. The system uses observed volatility, order-flow imbalance, liquidity regime, and price divergence signals to set pool-specific fee levels.

The design deliberately separates responsibilities. Estimation and policy updates run off-chain, where richer computation is cheap. On-chain logic is minimal and deterministic, enforcing bounded fee updates and per-swap overrides within Uniswap v4 rules.

## 1. Motivation

In concentrated-liquidity AMMs, LP performance is not only a function of traded volume. A useful approximation is:

$$
\mathrm{LPNet}(T) \approx \sum_{t=1}^{T} \phi_t V_t \,\Delta t \;-\; \mathrm{InventoryRisk}(T) \;-\; \mathrm{AdverseSelection}(T)
$$

Here, $\phi_t$ is the LP fee at time $t$, and $V_t$ is traded volume intensity. With static fees, LPs can be under-compensated during volatile or toxic-flow periods, then overprice execution during calm periods. APM addresses this mismatch by making fee selection state-dependent.

## 2. Protocol Constraints That Drive the Design (Uniswap v4)

APM is built around non-negotiable protocol facts:

1. A pool's dynamic-fee capability is fixed at creation and cannot be switched later.
2. Per-swap fee override is available in beforeSwap for dynamic-fee pools.
3. The beforeSwap interface must match v4 and return selector, delta, and fee override tuple.
4. Fee override is honored only when the override flag (0x400000) is set and the fee value is valid.
5. LP fee units are hundredths of a bip (1e-6), with maximum 1,000,000 (100%).
6. In hook execution, msg.sender is PoolManager; the original caller is provided separately.
7. Hook permissions are encoded in the deployed hook address bits.
8. A single hook can serve many pools, so all fee state must be keyed per PoolId.
9. PoolManager.updateDynamicLPFee is callable only through the authorized hook path.

These constraints determine both data structures and operational workflow.

## 3. System Architecture

APM is a closed-loop controller with four components:

- Plant: the Uniswap v4 pool state and swap engine.
- Actuator: the hook that enforces the fee seen by each swap.
- Controller: an off-chain service that computes target fees.
- Estimator: a data pipeline that converts on-chain observations into state features.

Operationally, the controller computes target fees periodically, applies guardrails, submits updates on-chain, and monitors outcomes.

## 4. Mathematical Formulation

For each pool i at time t, the control objective is:

$$
\max_{\pi_\theta}\; \mathbb{E}\!\left[\mathrm{FeeRevenue} - \mathrm{RiskCosts} - \mathrm{FlowLossPenalty}\right]
$$

subject to:

- $\phi_{\min} \leq \phi_{i,t} \leq \phi_{\max}$
- $|\phi_{i,t+1} - \phi_{i,t}| \leq \Delta_{\max}$
- update frequency limits via cooldown windows

A practical policy family is:

$$
\phi_t^{\mathrm{raw}} = \phi_0 + \alpha \sigma_t^\beta + \gamma \rho_t + \kappa d_t
$$

$$
\phi_t^{\mathrm{target}} = \operatorname{clip}\!\left(\phi_t^{\mathrm{raw}}, \phi_{\min}, \phi_{\max}\right)
$$

$$
\phi_{t+1} = \operatorname{clip}\!\left(\phi_t + \operatorname{clip}\!\left(\phi_t^{\mathrm{target}}-\phi_t,\; -\Delta_{\max},\; \Delta_{\max}\right),\; \phi_{\min},\; \phi_{\max}\right)
$$

Where:

- $\sigma_t$: realized volatility over a rolling window
- $\rho_t$: flow imbalance proxy
- $d_t$: external-price vs pool-price divergence proxy

This formulation is simple enough for production tuning, while still capturing first-order market regime shifts.

## 5. On-Chain Design (No Heavy Logic in Swap Path)

The hook keeps per-pool fee state. At minimum, each PoolId has current fee, min/max fee, maximum step size, cooldown, last update timestamp, and enable/disable status. Updater authorization is separate from admin control.

The update endpoint enforces three checks before writing a new fee:

1. bounds check against min/max
2. cooldown check against last update
3. step-size check against max allowed move

In beforeSwap, the hook performs constant-time reads and returns the per-swap override fee with the required override flag set. If a pool is disabled or stale, the hook should fall back to a safe bounded fee. No loops, external price calls, or expensive math should run in this path.

Optionally, APM can sync the same value to the pool's dynamic fee baseline through PoolManager.updateDynamicLPFee, but only when pool configuration and caller permissions satisfy v4 requirements.

## 6. Off-Chain Controller Responsibilities

The controller handles data collection, feature computation, policy evaluation, guarded fee selection, and transaction submission. It should treat chain reorgs and delayed inclusion as first-class concerns, with explicit finality rules and retry policy.

A typical control cycle is: pull latest state, compute features, generate target fee, clamp by risk limits, submit update transaction, then record realized outcomes for parameter refresh.

## 7. Security and Operational Safety

APM assumes adversarial conditions and should be operated accordingly.

Access control: updater keys and admin keys should be separated. Production deployments should use multisig governance and controlled key rotation.

Manipulation resistance: windowed estimators reduce one-block sensitivity. Max step and cooldown controls reduce oscillation and griefing surface. Inputs should be freshness-checked, and external reference feeds should be robust and monitored.

Caller semantics: policy logic must not assume msg.sender is the end user. In hooks, sender identity usually reflects routers or aggregators, so any sender-based policy must include trusted decoding and validation rules.

Failure handling: if the controller goes offline, the last valid fee remains active. Emergency controls should support clamp, freeze, and updater revocation paths.

## 8. Gas and Performance

The swap path is intentionally lightweight: one state read, one bounded fee selection, one override return. Expensive computation stays off-chain. This separation preserves determinism and keeps gas overhead predictable.

## 9. Product and Integration Risks

Technical correctness is necessary but not sufficient. Hooked pools still depend on routing and distribution. Uneven router support, fragmented aggregator behavior, and direct competition from other dynamic-fee hooks can limit realized flow.

APM should therefore ship with an integration plan, not only a control policy. That plan should cover router partnerships, measurable execution quality reporting, and LP-facing performance analytics.

## 10. Delivery Plan

Phase 1: single-pool deployment, conservative rule-based policy, strict bounds, and observability.

Phase 2: adaptive parameter updates, richer toxicity proxies, and coordinated multi-pool control.

Phase 3: formal invariant checks, external security audit, and stress testing under reorg and latency scenarios.

## 11. Non-Goals

APM does not attempt to run on-chain machine learning, replace routing infrastructure, or guarantee positive LP outcomes in all regimes.

## 12. References

- Uniswap v4 Dynamic Fees: https://developers.uniswap.org/docs/protocols/v4/concepts/dynamic-fees
- Uniswap v4 Hooks Concepts: https://developers.uniswap.org/docs/protocols/v4/concepts/hooks
- Uniswap v4 Swap Hooks Guide: https://developers.uniswap.org/docs/protocols/v4/guides/hooks/swap-hooks
- Accessing msg.sender in v4 Hooks: https://developers.uniswap.org/docs/protocols/v4/guides/hooks/accessing-msg.sender
- Uniswap v4 Deployments: https://developers.uniswap.org/docs/protocols/v4/deployments
- v4 Core IHooks.sol: https://github.com/Uniswap/v4-core/blob/main/src/interfaces/IHooks.sol
- v4 Core LPFeeLibrary.sol: https://github.com/Uniswap/v4-core/blob/main/src/libraries/LPFeeLibrary.sol
- v4 Core PoolManager.sol: https://github.com/Uniswap/v4-core/blob/main/src/PoolManager.sol
- LVR background paper: https://arxiv.org/abs/2208.06046
