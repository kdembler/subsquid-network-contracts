# Smart Contract Initiatives

This document summarizes the current and proposed smart-contract initiatives that need ownership decisions. Each initiative is framed around goal, status, risks, recommendation, and immediate next steps.

## Summary

| Initiative                                              | Status                                                                                   | Recommendation                                                                                          | Priority        |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | --------------- |
| Network contract migration                              | Spec and implementation branch exist; not yet production-ready.                          | Finish. This is the strategic contract workstream.                                                      | Critical        |
| Merkle rewards distribution                             | Contract/backend work is on `dev`; PR to `main` is open; hardening still needed.         | Finish, but only after security, testing, and cutover gaps are closed.                                  | High            |
| Direct operator portal access                           | No spec yet; external demand unconfirmed; main near-term need may be internal.           | If scoped to internal use, low-effort and ship without external audit; otherwise specify first.         | Medium          |
| Tethys/testnet redeployment                             | Existing testnet appears operational, but admin ownership is unclear.                    | Treat as an enabling dependency if current keys are unavailable.                                        | High dependency |
| Cross-chain portal pool staking / frictionless payments | RFC exists; mostly product, UX, payments, and integration work around live portal pools. | Defer from short-term contract roadmap.                                                                 | Low             |
| Cross-chain portal pools                                | Idea-level only; no spec or implementation. SQD already exists on Base and BNB.          | Defer; revisit only if multi-chain native pool deployment is genuinely required.                        | Low             |
| Solana token                                            | Draft PR #78 adds an isolated SPL mint utility (10-decimal); no bridge to EVM SQD.       | Drop or defer unless there is a confirmed external requirement.                                         | Low             |
| SP1 fraud-proof onchain verifier (Snoopy)               | Contracts exist in `sp1-onchain-verifier` repo; not audited; no slashing wiring.         | Note as possible workstream; bounded scope, likely below migration / rewards priority.                  | Low             |

## 1. Network Contract Migration

Links:

- Spec: [SQD Network Migration Phase 2 Technical Details](https://github.com/subsquid/specs/blob/sqd-migration/network-rfc/migration/Phase2.md)
- Branch: [`feat/migration`](https://github.com/subsquid/subsquid-network-contracts/tree/feat/migration)

### Goal

Move the live SQD Network contracts to a V2 system that supports RZAI/token migration, upgradeability, safer contract patterns, higher worker counts, on-chain worker tiers, and future protocol features.

The migration is not just a token replacement. It is the point where the network can move from rigid, mostly non-upgradeable contracts to an architecture that can evolve without full redeployments for every major protocol change.

### Current Status

- A detailed Phase 2 migration spec exists.
- An implementation branch exists on top of `dev`.
- The branch adds V2 versions of the core contracts, including router, network controller, staking, worker registration, rewards, gateway registry, vesting/holding factories, and reward strategies.
- The implementation follows the broad direction in the spec: UUPS proxies, initializer-based contracts, pausable/admin roles, storage gaps, `SafeERC20`, bounded batch operations, and more scalable worker/reward accounting.
- Deployment scripting exists, but the branch still needs to be turned into a clean reviewable PR.
- No complete production migration package is visible yet. The missing package is the combination of tests, deployment rehearsals, migration runbook, frontend/indexer plan, operational access plan, and audit scope.

### Value

- Unlocks the RZAI/token migration.
- Removes current protocol rigidity caused by immutable SQD references and non-upgradeable contracts.
- Enables on-chain worker tiers as part of worker registration, which is difficult to add cleanly to the current V1 contracts.
- Creates a safer base for future changes such as inflation, gateway changes, or reward-model updates.
- Addresses scaling issues around worker sets and reward calculation.
- Reduces future migration cost by moving the system behind upgradeable proxy addresses.

### Risks

- This is a full-system migration rather than an isolated contract upgrade.
- Workers, stakers, gateways, vesting users, temporary-holding users, backend services, indexers, and frontends all need a coherent transition path.
- Some user funds may remain tied to legacy vesting/temporary-holding flows until unlock or vesting events happen.
- Old and new network contracts may need to run in parallel for a transition period.
- Reward continuity needs explicit rules: final legacy rewarded block, first V2 reward range, claim behavior, and any seeded state.
- Upgrade/admin/pauser roles must be assigned correctly before launch.
- The branch needs dedicated V2 tests, storage-layout review, deployment rehearsal, and external audit.

### Recommendation

Finish this initiative. It is the highest-value smart-contract workstream and should be treated as the strategic roadmap item.

Do not commit to a mainnet launch date until the migration path is rehearsed end to end. The near-term deliverable should be a reviewed implementation plan and Linear breakdown, not the migration itself.

One important caveat: V2 is not a hard necessity if the RZAI token migration is delayed or descoped. In that scenario the existing contracts could continue running and worker tiers could be implemented as an off-chain layer instead. V2 is still the correct long-term direction and the recommendation stands, but the urgency depends on RZAI timing.

### Immediate Next Steps

1. Clean `feat/migration` into one or more reviewable PRs.
2. Remove or intentionally isolate deployment broadcast artifacts.
3. Add V2-focused tests for initialization, upgrades, roles, worker migration, staking, reward calculation, reward distribution, gateway bounds, vesting/holding behavior, and deployment scripts.
4. Decide the migration model: fresh state, seeded state, or hybrid.
5. Define the old/new reward boundary and claim behavior.
6. Prepare testnet rehearsal and audit scope before mainnet planning.

## 2. Merkle Rewards Distribution

Links:

- Open PR: [#72 Introduce new backend](https://github.com/subsquid/subsquid-network-contracts/pull/72)
- Merged contract PR: [#67 Merkle Tree-based Distributed Rewards Distribution System V2](https://github.com/subsquid/subsquid-network-contracts/pull/67)
- Merged backend PR: [#70 Adding NestJS backend for automated rewards calculation and merkle tree distribution](https://github.com/subsquid/subsquid-network-contracts/pull/70)

### Goal

Replace direct per-worker reward submission with a scalable Merkle-root distribution flow.

The backend calculates rewards for a block range, builds a Merkle tree, stores distribution artifacts, commits the root on-chain, supports approval, and distributes rewards in bounded batches.

### Current Status

- The Solidity reward distribution contract has been merged into `dev`.
- The NestJS rewards backend has been merged into `dev`.
- PR #72 is open from `dev` to `main`.
- `main` has since received the portal-contracts import, so the PR likely needs refresh/rebase before landing.
- The backend covers reward calculation, root generation, distribution lifecycle, storage, reporting, and admin operations.
- Several production-readiness items remain unresolved around admin security, persistence, recovery, signing, and test coverage.

### Value

- Makes reward distribution more scalable as worker counts grow.
- Reduces on-chain payloads by committing Merkle roots instead of submitting large per-worker reward sets.
- Gives the team a clearer operational model for calculating, approving, distributing, and recovering reward batches.
- Can potentially be delivered before the full network migration if the cutover is isolated.

### Risks

- Admin endpoints currently need proper protection before the backend is exposed outside a trusted environment.
- Some active distribution state is in memory, which is fragile across restarts or multiple backend instances.
- Recovery paths are incomplete or placeholder-level in some areas.
- Signing needs to be finalized around Fordefi or another approved production setup; raw distributor private keys in environment variables are not a sufficient long-term operational model.
- Contract/backend compatibility needs stronger end-to-end tests using real generated proofs.
- Cutover must avoid skipped or double-paid block ranges.
- The work may overlap with the broader migration; sequencing needs an explicit decision.

### Recommendation

Finish this initiative, but do not treat it as production-ready yet.

This is valuable and closer to delivery than the migration, but the remaining issues are operational/security issues rather than polish. It should either launch as a carefully isolated rewards-system upgrade or be bundled into the broader V2 migration if timelines align.

### Immediate Next Steps

1. Refresh PR #72 against current `main`.
2. Protect admin endpoints and define the intended access model.
3. Make backend state resilient to restarts and multi-instance operation.
4. Finalize signer/Fordefi setup and distributor quorum.
5. Add end-to-end tests for root generation, proof verification, commit, approval, batch distribution, restart, duplicate prevention, and recovery.
6. Define the production cutover runbook: last legacy block, first Merkle range, rollback behavior, and monitoring.

## 3. Direct Operator Portal Access

Link:

- Import PR: [#77 Import sqd-portal-contracts into packages/portal-contracts](https://github.com/subsquid/subsquid-network-contracts/pull/77)

### Goal

Add a new portal access path where portal operators stake their own SQD directly instead of relying on third-party users to stake SQD into portal pools in exchange for rewards.

This should be treated as a complement to the existing live portal-pool system, not as a replacement for it.

### Current Status

- Existing portal pools have already been deployed and are live.
- The imported portal contracts include pool factory, pool implementation, registry, fee routing, liquid portal tokens, whitelist support, credit/debt reward accounting, and exit queue logic.
- This specific initiative is the new direct operator staking path.
- I did not find a concrete branch, PR, spec, or implementation for this direct operator path.
- External operator demand is not yet confirmed; the more concrete near-term need may be for our own internal portal operations.
- The open design question is whether this should be a new contract, an extension of the current portal contracts, or a simpler operational/UI mode using part of the existing system.

### Value

- Gives portal operators a simpler route to access if they already hold enough SQD and do not want to attract external stakers.
- Reduces dependence on third-party liquidity and reward incentives for operator onboarding.
- Could make portal onboarding faster and easier to explain.
- May reduce user-facing complexity for operators who only need to stake their own capital.
- If scoped only for internal use, the implementation is expected to be relatively low-effort and could ship without an external audit, since the contract would not be exposed to third parties.

### Risks

- The direct path must not disrupt the already-live portal-pool system.
- It may duplicate existing portal-pool mechanics unless the desired operator flow is specified clearly.
- Contract changes near the live portal system need careful audit and migration planning.
- If implemented as an extension to current pool contracts, contract-size and upgrade constraints may become relevant.
- If implemented as a separate contract, the system needs clear accounting and UI treatment so users understand the difference between pooled and self-staked portal access.
- The value depends on actual operator demand and whether operators are willing to lock their own SQD directly.

### Recommendation

Treat this as a focused product/contract extension to the live portal system.

Do not start contract implementation until there is a short spec explaining the operator flow, required stake/accounting model, whether existing portal contracts can be reused, and how this should appear in the portal UI.

If we decide to scope this for internal use only, the bar is much lower: contract changes can be smaller and shipping without an external audit is acceptable, since the surface area is limited to our own operators.

### Immediate Next Steps

1. Confirm the intended operator journey for direct access.
2. Confirm whether direct access should reuse existing portal-pool contracts, extend them, or use a separate contract.
3. Define how operator stake, rewards, exits, fees, and accounting differ from the live pooled model.
4. Confirm expected UI changes for distinguishing pooled portal staking from self-staked operator access.
5. Write a one-page spec and implementation approach before contract work starts.

## 4. Tethys / Testnet Redeployment

### Goal

Ensure there is a controllable staging network for migration, rewards distribution, portal contracts, network app, cloud integrations, and operational rehearsals.

### Current Status

- The current testnet appears to be in use, but admin ownership is unclear.
- The working assumption in the draft is that Tethys admin keys may have been lost some years ago.
- The repo contains deployment addresses for mainnet and testnet environments.
- The reward stats app points at an Arbitrum Sepolia deployment.
- There is no clear source artifact confirming current admin ownership for the existing Tethys/testnet deployment.

### Value

- A controlled testnet is required before safely rehearsing migration, rewards cutover, portal deployments, and frontend/indexer changes.
- If the current environment is not administrable, redeployment is the cleanest way to unblock future contract work.
- Owning testnet deployment also gives the team a repeatable runbook for future staging environments.

### Risks

- If keys are unavailable, admin-level testing on the existing deployment is not reliable.
- Redeploying can break consumers unless all addresses and environment variables are updated consistently.
- Network app, cloud, worker setup, scheduler, rewards backend, docs, dashboards, monitoring, and deployment JSON may all depend on current addresses.
- A partial redeploy without coordinated consumers could create a confusing staging environment.

### Recommendation

Treat this as an enabling dependency rather than a standalone product initiative.

If Janusz can confirm we still control the current testnet admin keys, avoid unnecessary redeployment. If not, redeploy a controlled staging environment before migration or rewards launch rehearsals.

### Immediate Next Steps

1. Confirm exact current admin/key status with Janusz.
2. Inventory all services and apps that use current Tethys/testnet addresses.
3. Decide recover vs redeploy.
4. If redeploying, create a complete address/env update plan before changing consumers.
5. Move testnet admin/upgrader/distributor ownership into the approved multisig/Fordefi/secret-management setup.

## 5. Cross-Chain Portal Pool Staking / Frictionless Payments

Link:

- RFC: [Frictionless Payments for Revenue Pools](https://github.com/subsquid/specs/blob/tokenomics-2.1/network-rfc/13_frictionless_payments.md)

### Goal

Reduce friction for users entering revenue/portal pools through embedded wallets, wallet-as-a-service, on-ramping, card or Apple Pay flows, and chain/asset routing.

### Current Status

- An RFC exists.
- The proposal is primarily product, UX, payments, onboarding, and integration work.
- I did not find a contract implementation in this repository.
- The specific smart-contract changes required are not yet defined.

### Value

- Could materially improve conversion into the live portal pool product.
- Could reduce the need for users to understand wallet setup, bridging, gas, and chain selection.
- Sits naturally at the intersection of UI/UX ownership and portal pool growth strategy.

### Risks

- Adds provider, custody, on-ramp, compliance, support, and failure-mode complexity.
- The primary metric is user conversion, not protocol capability.
- Cross-chain deposits can introduce bridging/accounting complexity that may spill into contracts.
- Starting contract work before the product flow is committed would likely waste effort.

### Recommendation

Defer from the short-term smart-contract roadmap.

Keep this as product/UX discovery attached to portal pool strategy. Revisit contract work only after the target flow, provider choices, chains, supported assets, and custody assumptions are clear.

### Immediate Next Steps

1. Keep this outside the active smart-contract implementation queue.
2. Clarify whether easier cross-chain/fiat entry is a current priority for portal pool growth.
3. If yes, define the user journey and provider stack first.
4. Only then identify whether contract changes are actually needed.

## 6. Cross-Chain Portal Pools

### Goal

Deploy portal pool contracts natively on other chains (for example Base) with cross-chain messaging back to the Arbitrum control plane, so users can stake into portal pools without bridging SQD to Arbitrum first.

This is distinct from initiative 5: that initiative is about easier UX for entering the existing Arbitrum-resident pools, while this is about deploying actual pool instances on other chains.

### Current Status

- No spec, branch, or implementation found in this repository.
- The SQD token already exists on Base (`0xd4554bea546efa83c1e6b389ecac40ea999b3e78`) and BNB Smart Chain (`0xe50e3d1a46070444f44df911359033f2937fcc13`), so the prerequisites for a native deployment exist on at least two non-Arbitrum chains.
- Idea-level only; no committed design choice on messaging stack or accounting model.

### Value

- Eliminates the bridging step for Base/BNB SQD holders who want to stake into portal pools.
- Could grow portal pool TVL by reaching liquidity that does not naturally route to Arbitrum.
- Lets the team match portal pool deployments to where target users already are.

### Risks

- Cross-chain messaging adds bridge/relayer security and operational risk.
- Reward, fee, and exit accounting need to stay consistent across chains, which is significantly harder than running on a single chain.
- Pool implementation upgrades become harder when contracts run on multiple chains, especially with the existing beacon-proxy upgrade pattern.
- Likely overlaps with frictionless payments (initiative 5); the same underlying user need can sometimes be solved more cheaply by smoothing entry from other chains rather than deploying multi-chain pools.

### Recommendation

Defer. Note as a possible future extension to the portal pool system, not a short-term contract workstream.

If it resurfaces, decide first whether the user need is best solved by frictionless payments, multi-chain pool deployment, or neither.

### Immediate Next Steps

1. Keep outside the active smart-contract implementation queue.
2. If the question comes up, scope the user need first: are users actively asking for native Base/BNB pools, or is the underlying ask really easier entry from those chains?
3. Only if native multi-chain pools are clearly required, pick target chains, messaging stack, and accounting model before any contract work.

## 7. Solana Token

### Goal

Potentially create a Solana-native SQD token, bridge/wrapper, or Solana-side liquidity/payment primitive.

### Current Status

- A draft PR is open: [#78 Add Solana SQD SPL token package, mint CLI, and tests](https://github.com/subsquid/subsquid-network-contracts/pull/78). It adds `packages/solana-sqd-token` with an SPL mint CLI and tests.
- The PR notes that EVM uses 18 decimals while standard SPL stores amounts as `u64`; `1,337,000,000 * 10^18` does not fit, so the proposed Solana mint uses 10 decimals while keeping the same whole-token supply.
- No matching artifact was found in `subsquid/specs`.
- The PR is an isolated SPL mint utility; no bridge or onchain link to the EVM SQD token is implemented.
- The business driver is still unclear.

### Value

- Unclear without a specific external requirement.
- Possible drivers could include exchange support, liquidity, Solana-native payments, ecosystem integrations, or partner commitments.

### Risks

- Token issuance or bridging across Solana adds audit, bridge, custody, liquidity, support, and operational risk.
- It is not required for the current migration, rewards distribution, portal pool work, or testnet ownership.
- Without a confirmed driver, this would distract from higher-value contract work.

### Recommendation

Drop from the active roadmap unless leadership confirms a concrete external requirement.

If it resurfaces, start with business justification, legal/security review, bridge/custody design, and ecosystem requirements before implementation.

### Immediate Next Steps

1. Ask leadership/product whether there is a real requirement or just a historical idea.
2. If there is no committed requirement, close/defer it.
3. If there is a committed requirement, open a discovery task rather than an implementation task.

## 8. SP1 Fraud-Proof Onchain Verifier (Snoopy)

Links:

- Repo: [`subsquid/sp1-onchain-verifier`](https://github.com/subsquid/sp1-onchain-verifier)
- Off-chain prover: [`subsquid/snoopy`](https://github.com/subsquid/snoopy)
- Off-chain commitments: [`subsquid/assignments_commiter`](https://github.com/subsquid/assignments_commiter) (currently private)

### Goal

Add an onchain fraud-detection path for Subsquid network queries: workers that return responses inconsistent with peer consensus can be flagged via Succinct SP1 (Groth16) ZK proofs verified onchain, providing a basis for slashing.

The smart-contract scope is the three contracts in `sp1-onchain-verifier`:

- `CommitmentHolder` (UUPS proxy) — circular-buffer registry of worker assignment epochs (Merkle root + active timestamp + assignment ID), used to verify that the assignment set inside a proof matches what was active onchain.
- `FixedSP1VerifierWrapper` — wraps the SP1 Groth16 verifier and adds business-logic checks; decodes the public output (5 sample assignment-root/timestamp pairs plus the outlier peer ID and timestamp).
- `ProvingManager` (UUPS proxy) — entry point that stores named proof configurations and emits `FraudFound(peer_id, timestamp)` on successful verification.

### Current Status

- Contracts exist with deployment scripts (`script/00_deploy_ProvingManager.s.sol`, `01_grant_access.s.sol`, `02_upload_config.s.sol`) and a small test suite (`FixedTest.t.sol`, `WorkflowTest.t.sol`).
- The repo is independent of this monorepo; the off-chain prover (`snoopy`) and committer (`assignments_committer`) live in their own repos.
- Not yet audited.
- No integration with the live worker-registration / staking system: the verifier emits a `FraudFound` event but does not currently trigger slashing.

### Value

- Provides cryptographic enforcement of query correctness, complementing economic incentives.
- Closes a current gap where misbehaving workers have no onchain consequence.
- Limited blast radius if added carefully: the verifier can ship in a "detect only" mode (events only) before any slashing wiring is enabled.

### Risks

- ZK verification stacks (SP1 verifier, Groth16 precompile) need correct integration; bugs mean either false slashing or unenforceable proofs.
- Proof public-output format must remain consistent with what `snoopy` and `assignments_committer` produce; coupling across three repos introduces version-drift risk.
- Connecting `FraudFound` events to actual slashing on the network contracts is out of current scope and will need a separate design touching the migration / worker-registration work.
- Requires an external audit before any slashing path is activated on mainnet.

### Recommendation

Note as a possible workstream, not on the active short-term roadmap. The work is bounded: harden contracts, define the slashing integration path, audit, and deploy. Likely lower priority than migration and rewards distribution, but not blocked by them.

### Immediate Next Steps

1. Confirm whether onchain fraud detection is a current product priority. If not, leave as-is.
2. If yes: review contracts for completeness and edge cases, decide event-only vs slashing scope, line up audit, define the integration path with worker registration / staking (likely V2).
3. Decide whether to bring `sp1-onchain-verifier` into this monorepo (similar to the recent portal-contracts subtree import) or keep it standalone.

## Cross-Cutting Ownership Items

These are not separate product initiatives, but they are blockers or dependencies for taking ownership of the initiatives above.

### Operational Access

Required ownership checks:

- Mainnet admin/upgrader ownership.
- Testnet admin/upgrader ownership.
- Proxy admin ownership for legacy upgradeable contracts.
- UUPS upgrade ownership plan for V2.
- Pauser roles.
- Reward distributor addresses and signing process.
- Fordefi vaults, approvers, API credentials, and recovery process.
- Deployer wallets and funding process.
- GitHub Actions, Docker, Netlify, RPC, Arbiscan, S3/R2, ClickHouse, Redis, and backend environment access.

Recommendation:

Treat access confirmation as a mandatory handover task before Janusz leaves. Any missing access should become a tracked blocker against the relevant initiative.

### Linear Tracking

Suggested parent issues:

1. `SC-MIG`: Network contract migration.
2. `SC-RWD`: Merkle rewards distribution.
3. `SC-PORT`: Direct operator portal access.
4. `SC-TST`: Tethys/testnet redeployment.
5. `SC-BACKLOG`: Deferred/discovery initiatives.
6. `SC-HO`: Operational ownership and access handover.

Each issue should include goal, current state, linked branch/PR/spec, owner, risk level, required access, done criteria, and target date.
