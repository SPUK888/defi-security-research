# DeFi Security Research — 0xSPUK

Independent smart-contract security research on live DeFi bug-bounty targets.

I run an **autonomous audit pipeline** — continuous static analysis (Slither, Aderyn, Semgrep) across tracked bug-bounty programs, with a local-LLM triage engine that reasons over every finding, files evidence, and escalates real leads. When it surfaces something worth a human look, I take the target apart by hand and publish what I find.

This repo is the human half of that loop: **deep, honest writeups — one target at a time, weekly.**

Most of these are *negative results* — "no exploitable bug in scope, here's exactly why." That's deliberate. In real auditing the majority of rigorous work ends in a clean verdict, and the value is in the reasoning: what was checked, what looked exploitable but wasn't, and where a real bug *would* live if one existed. A thousand documented cleans are what make the one true anomaly obvious.

> **Approach note:** this is AI-augmented research. The pipeline is automated and I use LLMs as a force-multiplier for triage and code navigation — but every conclusion below is reasoned from the actual source and stated so it can be checked against the code. Corrections welcome via issues.

---

## Writeups

| Date | Target | Platform | Scope | Verdict |
|---|---|---|---|---|
| 2026-07-16 | [**1inch Aqua**](2026-07-16-1inch-aqua.md) | Immunefi | Shared-liquidity core (`Aqua`, `Balance`, `AquaApp`, `AquaRouter`) | No exploitable bug in core |
| 2026-07-16 | [**IPOR Protocol**](2026-07-16-ipor-protocol.md) | Immunefi | AMM pools, router, ipToken, governance | Scope exhausted — clean |
| 2026-07-16 | [**Moonwell**](2026-07-16-moonwell.md) | Code4rena | Compound-v2 / Aave-stkToken fork | 16 high-sev flags triaged — all FP/low |

*New writeup every week. Methodology is consistent across all of them (below).*

---

## Methodology

1. **Automated pass** — Slither + Aderyn + Semgrep (Decurity solidity rules), scoped to in-scope production code (dependencies, tests and mocks filtered out).
2. **Triage** — every high-severity flag is judged against a maintained library of false-positive signatures (proxy delegatecall, gated `transferFrom`, initializer variables, 0.8 arithmetic, etc.) and real-bug signatures (attacker-controlled delegatecall, missing CEI, spot-price oracles, unguarded `initialize()`). Flags that survive get escalated.
3. **Manual review** — for escalated targets (and always for the high-value ones) I read the actual source: custody paths, access control, checks-effects-interactions ordering, arithmetic/rounding, and cross-contract composition.
4. **Verdict + evidence** — a documented conclusion with the reasoning, filed and re-checked when the target ships new code.

## Contact
0xSPUK · AI-Augmented Security Automation Engineer · Independent DeFi & smart-contract security research
