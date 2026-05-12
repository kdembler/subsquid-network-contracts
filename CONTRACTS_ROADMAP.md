# SQD Network Contracts — Roadmap & Recommendations

## Bottom line

The strategic contract workstream over the next quarter is the **network contract migration**: a re-platforming that unlocks the RZAI token migration and removes the cost of future protocol changes. Everything else is opportunistic, future work, or should be deferred or dropped.

## Priority order

1. **Network contract migration** — strategic, multi-month, audit required.
2. **Merkle rewards distribution** — good to ship when capacity allows; not urgent at current worker counts. Best bundled with the migration if timelines align.
3. **Direct operator portal access (internal scope)** — small, opportunistic build if internally needed.
4. **SP1 fraud-proof verifier** — future workstream; sequence after migration.
5. **Defer / drop**: frictionless payments, cross-chain portal pools, Solana token.

## Initiatives

### Network contract migration — Continue (top priority)

**What.** Re-platform the live network contracts to a V2 system that supports the RZAI token migration, future protocol upgrades without redeployment, higher worker counts, and on-chain worker tiers as part of worker registration.

**Why it matters.** Today the protocol is rigid: any non-trivial change requires another migration. V2 unlocks the RZAI token migration, enables on-chain worker tiers as a first-class feature of worker registration, and dramatically reduces the cost of every protocol change after that.

**Cost / effort.** Multi-month. An implementation branch exists but needs more work, deployment rehearsal, and an external audit. Coordination required across workers, delegators, and network.

**Risk.** Highest blast radius of any item — a full-system migration with user funds in flight, reward continuity to preserve, and parallel old/new operation during cutover. Cannot be rushed.

**Recommendation.** Treat as the strategic roadmap item. Do not commit to a public launch date until rehearsed end-to-end. Allocate audit budget.

One important caveat: V2 is not a hard necessity with the RZAI token migration being delayed. In that scenario the existing contracts could continue running and worker tiers could be implemented as an off-chain layer instead. V2 is still the right long-term direction, but the urgency is tied to RZAI timing — it is a strong recommendation, not a blocking requirement.

### Merkle rewards distribution — Worthwhile, not urgent

**What.** Replace direct per-worker reward submission with a Merkle-root distribution flow. The backend computes rewards, posts roots onchain, and distributes in batches.

**Why it matters.** A cleaner long-term operational model and a path to scaling with higher worker counts. The current setup is somewhat hacky but works at today's worker count, and there is no near-term plan to grow that count materially. The pressure to ship is therefore low.

**Cost / effort.** Likely weeks of dev work outside of audit. Initial implementation done almost a year ago but requires hardening and testing. Additionally an audit and cutover runbook.

**Risk.** Mostly operational. Cutover must avoid skipped or double-paid reward ranges. Rushing the cutover would create more risk than the current system carries today.

**Recommendation.** Treat as flexible. The most efficient path is to bundle this into the V2 migration so it ships under a single cutover and a single audit. Stand-alone delivery is also viable, but only worth doing if there is a clear capacity window and migration timing has not yet locked in.

### Direct operator portal access — Build small if internally needed

**What.** Allow portal operators to stake their own SQD directly, instead of relying on third-party stakers via the existing pool flow.

**Why it matters.** External operator demand is unconfirmed. The clearer near-term need is for our own internal portal operations.

**Cost / effort.** Low if scoped to internal-only use — a small contract change, no external audit needed because the surface is limited to us. Higher if positioned as a public product, where it would need a spec, audit, and UI work.

**Risk.** Mostly opportunity cost. Scope creep here would compete with migration time.

**Recommendation.** If we need it internally, build the minimal version. Do not invest in a public-facing version without confirmed external demand.

### SP1 fraud-proof verifier — Future workstream, after migration

**What.** Onchain fraud detection: workers returning query responses inconsistent with peer consensus can be flagged via ZK proofs verified onchain, providing a basis for slashing. Contracts already exist in a separate repo; off-chain components are also in dedicated repos.

**Why it matters.** Closes a real gap — today, misbehaving workers face no onchain consequence. But it is not blocking any near-term protocol goal.

**Cost / effort.** Bounded. Contracts mostly exist; remaining work is hardening, an external audit, and integration with worker-registration / staking for actual slashing. The integration step likely touches the same surface as the V2 migration, which is why sequencing matters.

**Risk.** Doing this before migration risks rework once V2 lands.

**Recommendation.** Note as a future workstream. Plan to activate after migration is stable.

### Frictionless portal pool staking — Defer

**What.** Easier entry to portal pools through embedded wallets, on-ramps, fiat, and chain routing.

**Why it matters.** Conversion play, not protocol capability.

**Cost / effort.** Mostly product, UX, and integration work — minimal contract scope until the product flow is committed.

**Recommendation.** Defer unless strong need is identified and we want to optimize.

### Cross-chain portal pools — Defer

**What.** Deploy portal pool contracts natively on chains where SQD already exists (Base, BNB), with cross-chain messaging back to Arbitrum.

**Why it matters.** Could reach liquidity on chains where users already are — but the same UX outcome can usually be reached more cheaply via frictionless-payments-style entry, without deploying new pool instances.

**Cost / effort.** Significant. Multi-chain contracts, bridge or messaging risk, multi-chain accounting, harder upgrades.

**Recommendation.** Defer. Confirm whether the user need is real and not better solved by frictionless payments before any contract work.

### Solana token — Drop unless externally driven

**What.** A Solana-native SQD token. A draft PR adds an isolated SPL mint utility with no bridge to the EVM token.

**Why it matters.** No confirmed business driver. The current PR is small and self-contained; a real Solana token initiative (with a working bridge) is much larger.

**Cost / effort.** PR as drafted is small. A real bridge would carry audit, custody, liquidity, and ongoing operational cost.

**Recommendation.** Drop from the active roadmap unless we confirm a concrete external requirement.
