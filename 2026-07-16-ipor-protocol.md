# IPOR Protocol — Security Review of the AMM Pool & Router Core

**Target:** IPOR Protocol, Immunefi program `ipor` (`immunefi.com/bug-bounty/ipor/`)
**Scope reviewed:** Router proxy, AMM pools (USDT / USDC / weETH / stETH), ipToken, governance
**Date:** 2026-07-16 · **Verdict:** In-scope attack surface fully mapped — no submittable bug.

---

## 1. Scope discipline first

IPOR is an interest-rate-swap AMM. Before touching code I mapped the **paid** surface against the program's own scope page — this matters more than usual here because IPOR excludes some of the most tempting attack classes:

- **"Interest Rate Swaps opening/closing"** is excluded — which kills *any* finding in swap open/close/unwind/PnL-settlement, regardless of whether the bug is real.
- **"Liquidity of a pool equals zero"** is excluded — killing the classic first-depositor / inflation exchange-rate attack.
- **"IPOR index manipulation via AAVE & Compound"** is excluded — killing that specific oracle vector.
- Plasma Vault / IPOR Fusion, the Power-Token staking system, and the not-actually-deployed USDM pool are all out of scope (confirmed the USDM pool returns all-zero governance config on-chain — zero funds at risk).

So the real target is the **liquidity-provision path** (mint/redeem ipTokens against pool balance) and the **router's access control and reentrancy**, *not* the swap engine. Everything below stays inside that boundary.

## 2. The vector that looked exploitable — and the PoC that killed it

The most promising lead was in the stETH pool's ETH entry point. `AmmPoolsServiceStEth._depositEth()` credits liquidity based on
`stETH.balanceOf(address(this))` **after** the deposit, rather than the value returned by Lido's `submit()`. Combined with a `provideLiquidityEth` path that (on inspection) appeared to lack a `msg.value == ethAmount` check, the hypothesis was:

> A caller sends `ethAmount` as an argument but less actual `msg.value`; leftover ETH is stranded in the router and a later caller sweeps it — or the balance-delta accounting mis-credits shares.

This is exactly the kind of thing you do **not** conclude from reading alone. I built a **Foundry fork test** to actually execute it against mainnet state — and it **disproved the hypothesis**. The router's `_returnBackRemainingEth()` runs unconditionally at the end of *every* top-level call and refunds 100% of leftover native ETH to `msg.sender`. There is no window in which stray ETH sits in the contract to be stolen, and the balance-delta is taken across the full, refunded call. The hole closes by design.

**Lesson worth stating:** a `balanceOf`-based accounting pattern is a genuine smell, but "smell" is not "bug." The PoC is what separates a finding from a false alarm — this one caught a false alarm.

## 3. Liquidity-provision exchange rate

The other core concern is whether LPs can redeem more than their share — draining the pool. IPOR's newer `AmmPoolsServiceBaseV1._redeem()` / `_provideLiquidity()` price shares off `getLiquidityPoolBalance()`, which **nets out reserved swap collateral on every call**. So even though the reserved-collateral check looks different from the older legacy code's explicit guard, it achieves the same protection structurally: redemption can't pull the pool below what's reserved for open swaps. Verified the netting path end-to-end.

Swap PnL, though mostly out of scope, was checked for spillover into the LP path: `SwapLogicBaseV1.normalizePnlValue()` hard-caps payout at `swap.collateral` even at 1000× leverage, so `totalCollateralPayFixed` is a true upper bound on liability — the LP balance can't be surprised by an unbounded swap loss.

`AssetManagementLogic.calculateRebalanceAmountBeforeWithdraw` (the treasury/AMM rebalancing) I verified **algebraically**: post-withdrawal it always leaves `S·r + O ≥ O` in the treasury when the pool is solvent (S = assets minus withdrawal, r = target ratio, O = obligations). No path to draining the treasury via rebalance.

## 4. Access control & reentrancy

- **Router reentrancy** uses a *single global* transient-storage-style flag (`AccessControl._nonReentrantBefore`) shared across every guarded selector. I read `IporProtocolRouterEthereum.sol` **selector-by-selector** to confirm there's no guarded/unguarded mismatch — a shared global flag is only safe if *every* fund-moving selector actually sets it, and they do.
- **Governance** — every state-changing function in `AmmGovernanceServiceBaseV1.sol` is correctly gated: 12 via the router's single combined `_onlyOwner()` branch, and 2 (`transferToTreasury` / `transferToCharlieTreasury`) via correct in-function role checks.
- **ipToken mint/burn** — verified **live on-chain** with `cast call` simulation across the stETH / weETH / USDC ipTokens: `mint()` reverts for a random caller (`IPOR_327`) and succeeds only when simulated *from the router address*. This is the check I trust least on a read alone, so I confirmed it against deployed bytecode rather than source.
- The IPOR Oracle's access control is correct but **moot** for the paid surface: its index/IBT price feeds only swap-pricing logic, never the LP exchange-rate calc — so any oracle issue falls under the excluded swap category anyway.

## 5. Verdict

The in-scope paid attack surface — LP mint/redeem, router auth and reentrancy, governance, treasury rebalance — is **fully mapped and clean**. The one genuinely exploitable-looking lead (the stETH ETH-refund vector) was disproved with a fork-test PoC. This is consistent with the ~$3k already paid to prior researchers before this review: the easy and moderate bugs appear to be found and fixed.

**No submittable bug in the reviewed scope.**

**Methodology note:** live deployed source pulled via GitHub's contents API (`Accept: application/vnd.github.raw`) rather than trusting summaries; every on-chain claim cross-checked with `cast call` against mainnet; and the one exploitable hypothesis taken all the way to a Foundry fork-test before conclusion. Read-only summaries of Solidity lie — deployed bytecode and an executable PoC don't.
