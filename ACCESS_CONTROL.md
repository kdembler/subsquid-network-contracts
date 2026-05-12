# Access Control — Subsquid Network Production Contracts

This document inventories every privileged account on Subsquid's production deployments and what each can do. It is derived from on-chain reads of Arbitrum One, Ethereum, Base, and BNB Smart Chain (snapshot ~Apr 2026, Arbitrum block 457,895,969), reconciled against the contract source in this repo.

Production addresses are the ones listed in [`Readme.md`](./Readme.md). Where the on-chain state diverges from `packages/contracts/deployments/42161.json` or other repo artifacts, the on-chain state is treated as authoritative.

---

## Table of contents

1. [TL;DR](#tldr)
2. [Privileged principals](#privileged-principals)
3. [Arbitrum One — Network contracts](#arbitrum-one--network-contracts)
4. [Arbitrum One — Portal Pool system](#arbitrum-one--portal-pool-system)
5. [Token contracts](#token-contracts)
6. [Risks and observations](#risks-and-observations)
7. [Methodology](#methodology)

---

## TL;DR

- **No on-chain multisig in front of the network contracts.** Every principal that holds DEFAULT_ADMIN_ROLE / PAUSER_ROLE on a network contract is a bare EOA (empty bytecode). There is no Safe, no timelock, no on-chain governance.
- **One privileged address (`0xFa27…61Ee`) is an EIP-7702 delegated EOA.** It is co-admin of the active portal FeeRouter, and its installed delegate (`0x63c0…e32b`) reports `VERSION() = "1.3.0"`, consistent with a Safe-style smart-account. Practically this gives the address smart-account semantics, but the underlying EOA private key can revoke or replace the delegation at will, so the trust model is hybrid.
- **One non-EOA principal on a token: BNB SQD `minter()` is a proxy contract** (`0x37DCb4…322B`, ERC1967 proxy with implementation `0x072822da…bad1`). What it actually authorises depends on that implementation, which is not in this repo.
- **Two governance circles for the network/portal split.** The network contracts are administered by EOA `0x5800…`. The portal-pool contracts are administered by a different set of EOAs (`0x13378…`, `0x2a2fb…`, `0xc4233…`, plus operator `0x9FC9…`).
- **Deployer EOA still holds admin** on seven of the network contracts. It was revoked from Router and NetworkController but not from Staking, WorkerRegistration, RewardTreasury, DistributedRewardsDistribution, GatewayRegistry, VestingFactory, or TemporaryHoldingFactory.
- **DistributedRewardsDistribution** has no governance admin at all — only the original deployer EOA. If that key is lost, distributor whitelist and approval-threshold parameters can never change.
- **RewardTreasury admin can drain the contract directly** via `reclaimFunds()`, in addition to the indirect "whitelist a malicious distributor" route.
- **The README's FeeRouter is stale.** PortalPoolFactory currently points at `0x763a…BE6C`, not the README's `0x59c0…A787`. Its admins are the portal operator EOA `0x9FC9…` and the EIP-7702 delegated address `0xFa27…`, not the deploy-script set.
- **GatewayRegistry implementation has been upgraded** at least once since the initial deploy.

---

## Privileged principals

The "Account type" column reflects the chain state at the snapshot block. **EOA** means the address has no bytecode on its host chain. **EIP-7702 delegated EOA** means the address itself is an EOA, but it has installed a 7702 delegation that makes calls to it execute the delegate's code — the EOA's private-key holder retains the ability to revoke or replace the delegation. **Contract** means bytecode (typically a proxy) at the address.

| Label | Address | Account type | Scope of power |
|---|---|---|---|
| **Network deployer** | `0x1de89bbb2a41ead44b94556b8798471bd9ff4add` | EOA | DEFAULT_ADMIN_ROLE on Staking, WorkerRegistration, RewardTreasury, DistributedRewardsDistribution, GatewayRegistry, VestingFactory, TemporaryHoldingFactory. PAUSER_ROLE on the same set. VESTING_CREATOR_ROLE, HOLDING_CREATOR_ROLE. |
| **Network governance** | `0x5800eEB867D6c490051321239E15C3ac90e801C1` | EOA | DEFAULT_ADMIN_ROLE on Router, NetworkController, Staking, WorkerRegistration, RewardTreasury, GatewayRegistry, VestingFactory, TemporaryHoldingFactory. PAUSER_ROLE on the same set. **Owns the ProxyAdmin contracts** that authorise upgrades to Router and GatewayRegistry. |
| **Portal operator** | `0x9FC9Fc4cFD0993630cd0f37f00647d301EdCD2a9` | EOA | OPERATOR_ROLE on both live pools (Lambda × SQD and SQD Revenue Pool). DEFAULT_ADMIN_ROLE on PortalPoolFactory and on the active FeeRouter (`0x763a…`). POOL_DEPLOYER_ROLE on PortalPoolFactory. |
| **FeeRouter co-admin (7702)** | `0xFa27FdC303FA02F6F21Ec8F597421b7B34BD61Ee` | EIP-7702 delegated EOA → `0x63c0c19a282a1b52b07dd5a65b58948a07dae32b` (delegate reports `VERSION()` = "1.3.0", Safe-style) | DEFAULT_ADMIN_ROLE on the active FeeRouter (`0x763a…`). It deployed that contract and granted admin to itself, the deployer EOA, and `0x9FC9…`. |
| **Portal deployer** | `0x13378206434ff683b3d81d9d42b3e782fe4d9dbc` | EOA | DEFAULT_ADMIN_ROLE on PortalPoolFactory and PortalRegistry. PAUSER_ROLE on both. POOL_DEPLOYER_ROLE on PortalPoolFactory. |
| **POOL_DEPLOYER_1** | `0x2a2fbdef84219bdaa0c657e45447d6bdd7edaae2` | EOA | DEFAULT_ADMIN_ROLE on PortalRegistry. (Not on Factory.) |
| **POOL_DEPLOYER_2** | `0xc423362be9db384b79b7a8b21d68b65e3f1c63a7` | EOA | DEFAULT_ADMIN_ROLE on PortalPoolFactory and PortalRegistry. |
| **Vesting / rewards ops** | `0x31b53e955cb8776196c4029ac4e8ba796b3ff635` | EOA | VESTING_CREATOR_ROLE on VestingFactory. REWARDS_DISTRIBUTOR_ROLE on DistributedRewardsDistribution. |
| **Holding ops** | `0xeceb8a7d085764a13403a1e678ced91498478fa8` | EOA | HOLDING_CREATOR_ROLE on TemporaryHoldingFactory. |
| **Reward distributor #2** | `0x13e46e4488aa29f98465e1c76263fc4ff517b953` | EOA | REWARDS_DISTRIBUTOR_ROLE on DistributedRewardsDistribution. |
| **Reward distributor #3** | `0xf0be672978f394b2fcf34d0e27736eb92a93584a` | EOA | REWARDS_DISTRIBUTOR_ROLE on DistributedRewardsDistribution. |
| **BNB SQD owner** | `0x479Bc7bb7D90cE922F5997e797cb3B44844Ed3C6` | EOA (BNB) | `owner()` of the SQD token on BNB Smart Chain. |
| **BNB SQD minter** | `0x37DCb4E443a06A3FE0E7098519C1d81181E5322B` | Contract (ERC1967 proxy → `0x072822da16ad9b43d736840c86a1fb2f2870bad1`) | `minter()` of the SQD token on BNB Smart Chain. The implementation is not in this repo; what the minter contract actually permits, and who controls it, must be verified from the deployed bytecode. |

**Non-EOA principals (contracts holding roles):**

| Label | Address | Scope |
|---|---|---|
| Router ProxyAdmin | `0xfa6f03a3957cfa2e1df863127f9b411d1408ff44` | Authorises Router proxy upgrades. Owned by network governance EOA `0x5800…`. |
| GatewayRegistry ProxyAdmin | `0x38dfb6193da8dd2d3cd7ee1e7980fa914a5b5c89` | Authorises GatewayRegistry proxy upgrades. Owned by network governance EOA `0x5800…`. |
| RewardTreasury contract | `0x237abf43bc51fd5c50d0d598a1a4c26e56a8a2a0` | Holds REWARDS_TREASURY_ROLE on DistributedRewardsDistribution (allows it to invoke `claim`). |
| DistributedRewardsDistribution contract | `0x4de282bD18aE4987B3070F4D5eF8c80756362AEa` | Holds REWARDS_DISTRIBUTOR_ROLE on Staking. |
| PortalPoolFactory contract | `0x18184740eBE24881355E33cec620C44E575F2C70` | `owner()` of PortalPoolBeacon; FACTORY_ROLE on every live pool. |
| EIP-7702 delegate target for `0xFa27…` | `0x63c0c19a282a1b52b07dd5a65b58948a07dae32b` | Code that runs when the FeeRouter co-admin EOA is called. Reports `VERSION()` = "1.3.0" — Safe-style smart-account semantics. The EOA private key can revoke or replace this delegation. |
| BNB SQD `minter` proxy | `0x37DCb4E443a06A3FE0E7098519C1d81181E5322B` (BNB) | ERC1967 proxy. Implementation: `0x072822da16ad9b43d736840c86a1fb2f2870bad1`. Source not in this repo; implementation must be inspected separately to determine actual mint authority and admin chain. |

---

## Arbitrum One — Network contracts

### Router — `0x67F56D27dab93eEb07f6372274aCa277F49dA941`

- **Pattern:** TransparentUpgradeableProxy + AccessControl.
- **Implementation:** `0x4a7c41397f623ca04b60a59bcaa77346aeae86aa`.
- **Upgrade authority:** ProxyAdmin `0xfa6f03…ff44`, owned by network governance EOA `0x5800…`.
- **DEFAULT_ADMIN_ROLE holders:** `0x5800…` (deployer was granted at init, then revoked).
- **PAUSER_ROLE holders:** `0x5800…`.
- **What admin can do:** repoint references to WorkerRegistration, Staking, RewardTreasury, NetworkController, RewardCalculation. In effect, repoint every other piece of the network through this single contract.

### NetworkController — `0xf5462EF65Ca8a9Cca789c912Bc8ada80b582d68d`

- **Pattern:** AccessControl, non-upgradeable.
- **DEFAULT_ADMIN_ROLE holders:** `0x5800…` only (deployer revoked).
- **What admin can do:** set epoch length, lock period, bond amount, storage-per-worker, target capacity, yearly reward cap coefficient, staking deadlock window, and the whitelist of targets that vesting/holding contracts may call (`setAllowedVestedTarget`).
- **Note:** README's address is current. `42161.json` still lists an older NetworkController at `0x4cf580…` — the contract was redeployed; the deployments JSON is stale.

### Staking — `0xb31a0d39d2c69ed4b28d96e12cbf52c5f9ac9a51`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE holders:** **deployer `0x1de8…` and governance `0x5800…`** (both still active).
- **PAUSER_ROLE holders:** deployer + governance.
- **REWARDS_DISTRIBUTOR_ROLE holders:** DistributedRewardsDistribution contract `0x4de2…`.
- **What admin can do:** set max delegations per staker; set epoch lock duration. Pauser freezes all stake/withdraw/claim.

### WorkerRegistration — `0x36e2b147db67e76ab67a4d07c293670ebefcae4e`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE / PAUSER_ROLE holders:** deployer `0x1de8…` and governance `0x5800…`.
- **What admin can do:** the contract has no admin-gated configuration setters; only the inherited PAUSER_ROLE matters here. Pause freezes worker registration, deregistration, and bond withdrawal.

### RewardTreasury — `0x237abf43bc51fd5c50d0d598a1a4c26e56a8a2a0`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE / PAUSER_ROLE holders:** deployer `0x1de8…` and governance `0x5800…`.
- **What admin can do:**
  - **`reclaimFunds()`** (`RewardTreasury.sol:77`, `onlyRole(DEFAULT_ADMIN_ROLE)`): transfers the entire `rewardToken` balance directly to `msg.sender`. **A single admin call drains the treasury to the admin's own address.**
  - `setWhitelistedDistributor` (`onlyRole(DEFAULT_ADMIN_ROLE)`): whitelist arbitrary distributor contracts. A malicious distributor whitelisted by admin can also drain via `claimFor`.
  - Pauser halts all claims (does not protect against `reclaimFunds`, which has no `whenNotPaused` check).

### DistributedRewardsDistribution — `0x4de282bD18aE4987B3070F4D5eF8c80756362AEa`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE holders:** **deployer `0x1de8…` only.** Network governance `0x5800…` was never granted admin here.
- **PAUSER_ROLE holders:** deployer `0x1de8…` only.
- **REWARDS_DISTRIBUTOR_ROLE holders:** `0x13e46e…`, `0x31b53e…`, `0xf0be67…` — three EOAs.
- **REWARDS_TREASURY_ROLE holders:** RewardTreasury contract `0x237a…`.
- **What admin can do:** add/remove distributors (and via that, grant/revoke REWARDS_DISTRIBUTOR_ROLE), set required-approvals threshold, set committer window size, set round-robin block interval. The distributors collectively commit + approve reward distributions.

### GatewayRegistry — `0x8a90a1ce5fa8cf71de9e6f76b7d3c0b72feb8c4b`

- **Pattern:** TransparentUpgradeableProxy + AccessControlledPausableUpgradeable.
- **Implementation (current):** `0xa20ee6bd99b4da88652d6a7e0d013c96905adc58`. (The original `0x2591121…` from `42161.json` has been replaced; the contract has been upgraded at least once.)
- **Upgrade authority:** ProxyAdmin `0x38dfb6…5c89`, owned by `0x5800…`.
- **DEFAULT_ADMIN_ROLE / PAUSER_ROLE holders:** deployer `0x1de8…` and governance `0x5800…`.
- **What admin can do:** set the mana parameter (worker computation-unit economics), set average block time, set max gateways per cluster, set min stake, allow/disallow allocation strategies (and pick the default). Pause freezes registration, staking, allocations.

### VestingFactory — `0x1f8f83cd76baeca1cb5c064ad59203c82b4e4ece`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE / PAUSER_ROLE holders:** deployer `0x1de8…` and governance `0x5800…`.
- **VESTING_CREATOR_ROLE holders:** deployer `0x1de8…`, governance `0x5800…`, ops EOA `0x31b53e…`.
- **What creators can do:** deploy new SubsquidVesting contracts. Each vesting is owned by its beneficiary and constrained to call only `NetworkController`-whitelisted targets.

### TemporaryHoldingFactory — `0x14926ebf05a904b8e2e2bf05c10ecca9a54d8d0d`

- **Pattern:** AccessControlledPausable, non-upgradeable.
- **DEFAULT_ADMIN_ROLE / PAUSER_ROLE holders:** deployer `0x1de8…` and governance `0x5800…`.
- **HOLDING_CREATOR_ROLE holders:** deployer `0x1de8…`, governance `0x5800…`, ops EOA `0xeceb8a…`.
- **What creators can do:** deploy new TemporaryHolding contracts (each has its own immutable beneficiary, admin, and lock timestamp).

### RewardCalculation, SoftCap, EqualStrategy, SubequalStrategy, AllocationsViewer

No access control. RewardCalculation, SoftCap, AllocationsViewer are pure/view utilities. EqualStrategy is stateless. SubequalStrategy mutates state but only on behalf of `msg.sender` (no privileged role).

### Per-instance vesting / holding contracts

Created on demand by the factories above. Each Vesting is a `VestingWallet` whose owner is the beneficiary; ownership transfer is disabled (`_transferOwnership` reverts). Each TemporaryHolding has immutable `beneficiary`, `admin`, and `lockedUntil`. Both can only call out to network targets allowlisted by `NetworkController.setAllowedVestedTarget`, which is admin-controlled (governance EOA `0x5800…`).

---

## Arbitrum One — Portal Pool system

### PortalPoolFactory — `0x18184740eBE24881355E33cec620C44E575F2C70` (UUPS proxy)

- **Pattern:** ERC1967 proxy + UUPSUpgradeable + AccessControl.
- **Implementation (current):** `0xCe5D796769Ba065Bf61a8eFC892A8a835FAe0351`.
- **Upgrade authority:** anyone with DEFAULT_ADMIN_ROLE on the factory itself (`_authorizeUpgrade`).
- **DEFAULT_ADMIN_ROLE holders:** `0x9FC9…` (operator), `0x13378…`, `0xc4233…`.
- **PAUSER_ROLE holders:** `0x13378…` only.
- **POOL_DEPLOYER_ROLE holders:** `0x13378…`, `0x9FC9…`.
- **What admin can do:** **upgrade the beacon implementation, swapping the code of every live pool in one transaction**; change FeeRouter; change distribution-rate bounds; set min stake threshold and worker epoch length; manage the payment-token allowlist; toggle permissionless pool deployment; toggle the whitelist feature; pause global pool creation. Admin can also pause and `closePool()` on individual pools through the `onlyFactoryAdmin` hook in pools.

### PortalRegistry — `0x29edE9EB0ad3C02B6A98B0E41bF99Cd709812850` (UUPS proxy)

- **Pattern:** ERC1967 proxy + UUPSUpgradeable + AccessControl.
- **Implementation (current):** `0xC3725B2584Ad46c52f9eFA6F27d0291E3dbC3045`.
- **Upgrade authority:** DEFAULT_ADMIN_ROLE on the registry (UUPS).
- **DEFAULT_ADMIN_ROLE holders:** `0x13378…`, `0x2a2fb…`, `0xc4233…`. (Operator EOA `0x9FC9…` is **not** an admin here.)
- **PAUSER_ROLE holders:** `0x13378…` only.
- **What admin can do:** `setMinStake`, `setMana` (changes how computation units are calculated for indexer rewards), `setFactory` — i.e. swap which contract is allowed to register new clusters.

### PortalPoolBeacon — `0x16983f5a5816d4B04c92Ab43Fed3B2F212D4e568`

- **Pattern:** OpenZeppelin v5.0.0 `UpgradeableBeacon`, which inherits `Ownable`. Both `transferOwnership(address)` and `upgradeTo(address)` are `onlyOwner`.
- **Implementation (current):** `0x2981E64342fc76d531168Cf6754F81422138b3C4`.
- **`owner()`:** PortalPoolFactory `0x18184740…2C70`.
- **Current upgrade path (under the deployed factory implementation `0xCe5D7967…0351`):** the factory's `upgradeBeacon(address)` is the only function that calls into the beacon, and it only invokes `_beacon.upgradeTo(...)`. The factory does not expose a path to call `transferOwnership` on the beacon.
- **Latent escalation path:** the factory itself is UUPS-upgradeable, with `_authorizeUpgrade` gated by DEFAULT_ADMIN_ROLE on the factory (the three portal admin EOAs). Any one of those admins can deploy a new factory implementation that calls `_beacon.transferOwnership(...)` and then route ownership to an arbitrary address. So beacon ownership is *immutable in practice given the current code*, but *not immutable in principle* — it ultimately rests on the same three keys.

### Active FeeRouter — `0x763a0A80a815650281926b3174180d2f17cCBE6C`

This is the FeeRouter that the factory currently points at (`feeRouter()` → `0x763a…`). It is **not the address listed in the README**.

- **Pattern:** AccessControl, non-upgradeable.
- **DEFAULT_ADMIN_ROLE holders:**
  - `0x9FC9Fc4cFD0993630cd0f37f00647d301EdCD2a9` (portal operator EOA).
  - `0xFa27FdC303FA02F6F21Ec8F597421b7B34BD61Ee` — an EIP-7702 delegated EOA. This address deployed the contract and granted itself admin in the constructor; it later granted admin to the deployer EOA `0x1de8…` (since revoked) and to `0x9FC9…`. Both holders verified via `hasRole(0x00…, …)`.
- **Current configuration:** 8000 BPS to providers, 0 BPS to worker pool, 2000 BPS to burn. `workerPoolAddress` is `0x9FC9…`. `burnAddress` is the FeeRouter contract itself.
- **What admin can do:** change the fee split (must total 10000 BPS), change burn address, change worker pool address.

### Stale FeeRouterModule — `0x59c074ee3dd85125620B4A5b452C008Bc792a787`

This is the address the README calls `FeeRouterModule`. The factory no longer routes fees through it.

- **DEFAULT_ADMIN_ROLE holders (per deploy script):** original deployer + POOL_DEPLOYER_1 (`0x2a2fb…`) + POOL_DEPLOYER_2 (`0xc4233…`).
- **Current configuration:** 100% to providers, 0% to worker pool, 0% to burn — the original install. `workerPoolAddress` = `0x1291847E44A9144306CABA8B83504E1430C92E66`.
- **Effective state:** dead code. README needs an update.

### Lambda × SQD pool — `0x89ca93e09ec7355a1d6bd410fe0bb4c9b24542db`

- **Pattern:** BeaconProxy → PortalPoolImplementation; AccessControl per pool.
- **OPERATOR_ROLE holders:** `0x9FC9…`.
- **FACTORY_ROLE holders:** PortalPoolFactory `0x18184740…2C70`.
- **Whitelist:** disabled (`whitelistEnabled() == false`).
- **LPT receipt token:** `0xF7B057C3b0ee4dd101047B26Fd3964185B7d8cc4` (mint/burn locked to this pool by immutable `PORTAL_POOL`).
- **What operator can do:** topUpRewards, setDistributionRate (within factory bounds), setCapacity, transferOperator, manage whitelist and recover unclaimed rewards on FAILED/CLOSED pools.
- **What factory admins can do:** `pause()`, `unpause()`, and `closePool()` via the `onlyFactoryAdmin` hook (i.e. any of the three factory admin EOAs). Beacon upgrade replaces the entire pool logic.

### SQD Revenue Pool — `0x438c2a47e82cd445524ce5651ae7e6c1dd386d09`

Same access control structure as the Lambda pool.

- **OPERATOR_ROLE holders:** `0x9FC9…`.
- **FACTORY_ROLE holders:** factory `0x18184740…2C70`.
- **LPT receipt token:** `0x365709Ef4830B77a6EB4a689F13F57A1C22d8306`.

---

## Token contracts

### SQD on Ethereum — `0x1337420dED5ADb9980CFc35f8f2B054ea86f8aB1`

- ERC20 + ERC20Permit + ERC20Votes. No `owner()`, no roles.
- One-shot `registerTokenOnL2` for Arbitrum bridge registration; not callable after first invocation.
- Fixed supply: 1,337,000,000 SQD, minted at construction to a recipients list.

### SQD on Arbitrum One — `0x1337420dED5ADb9980CFc35f8f2B054ea86f8aB1`

- Bridged token (`SQDArbitrum`).
- Mint/burn restricted to the L2 gateway `0x096760F208390250649E3e8763348E783AEF5562` via `onlyL2Gateway`. Both the gateway and the L1 token address are immutable.
- No app-level admin.

### SQD on Base — `0xd4554bea546efa83c1e6b389ecac40ea999b3e78`

- `OptimismMintableERC20`-style bridge token.
- `BRIDGE` = `0x4200000000000000000000000000000000000010` (Base L2StandardBridge predeploy).
- `REMOTE_TOKEN` = `0x1337420dED5ADb9980CFc35f8f2B054ea86f8aB1` (Ethereum SQD).
- No app-level admin (no `owner()`, no DEFAULT_ADMIN_ROLE).

### SQD on BNB Smart Chain — `0xe50e3d1a46070444f44df911359033f2937fcc13`

- **`owner()` = `0x479Bc7bb7D90cE922F5997e797cb3B44844Ed3C6` (EOA).**
- **`minter()` = `0x37DCb4E443a06A3FE0E7098519C1d81181E5322B`.**
- The contract source is not in this repo. The exact powers of `owner` (e.g. ability to rotate the minter, change params) need to be verified against the deployed bytecode before this section is published.

---

## Risks and observations

1. **Single-key risk on most privileged roles.** Almost every principal that holds DEFAULT_ADMIN_ROLE on a production contract is a bare EOA, with no Safe, timelock, or on-chain governance in front. Compromise of `0x5800…` is sufficient to upgrade Router and GatewayRegistry, drain RewardTreasury directly via `reclaimFunds()` (or indirectly via a malicious distributor), and rewrite NetworkController parameters. Compromise of `0x9FC9…` is sufficient to control both live portal pools, the factory, and to co-administer the active FeeRouter — i.e. rewrite portal revenue routing and (with any of the three factory admins) upgrade every pool. The one exception is `0xFa27…61Ee`, which is an EIP-7702 delegated EOA — see point 8.
2. **RewardTreasury admin has direct drain authority.** `RewardTreasury.reclaimFunds()` (`packages/contracts/src/RewardTreasury.sol:77`) transfers the entire `rewardToken` balance to `msg.sender`, gated only by `onlyRole(DEFAULT_ADMIN_ROLE)`, with no `whenNotPaused`. Both the deployer EOA `0x1de8…` and the governance EOA `0x5800…` can invoke this in a single transaction. Pausing the contract does not prevent it.
3. **Deployer EOA still has admin on seven contracts.** It was revoked from Router and NetworkController but not from Staking, WorkerRegistration, RewardTreasury, DistributedRewardsDistribution, GatewayRegistry, VestingFactory, or TemporaryHoldingFactory. Either revoke from the rest or document the deployer key as an explicit recovery custodian.
4. **DistributedRewardsDistribution has only the deployer as admin.** Governance EOA `0x5800…` was never granted DEFAULT_ADMIN_ROLE here. If the deployer key is lost, the distributor allowlist and approval-threshold cannot be changed.
5. **README's FeeRouter is stale.** Factory points at `0x763a0A80…BE6C`. The README's `0x59c074…A787` is no longer wired in. The active FeeRouter is co-administered by `0x9FC9…` (operator EOA) and `0xFa27…` (EIP-7702 delegated EOA), neither of which is in the deploy script's intended admin set. Current split is 80/0/20, not the deploy script's 100/0/0. Update the README and confirm the configuration is intentional.
6. **GatewayRegistry implementation has been upgraded** (current `0xa20ee6…`, original `0x2591121…`). There is no in-repo record of when or why; track this in the docs going forward.
7. **BNB SQD has a privileged owner and a contract minter.** The owner `0x479Bc7…` is an EOA on BNB. The `minter()` `0x37DCb4…` is an ERC1967 proxy with implementation `0x072822da…bad1`, source not in this repo. Pull the implementation's verified source before publishing this doc externally; without it the actual mint authority on BNB is unknown.
8. **EIP-7702 hybrid principal.** `0xFa27FdC303FA02F6F21Ec8F597421b7B34BD61Ee` is one of two admins on the active FeeRouter. Its on-chain code is `0xef0100` followed by `0x63c0c19a282a1b52b07dd5a65b58948a07dae32b`, an EIP-7702 delegation pointer. Calls to it execute the delegate's code, which reports `VERSION()` = "1.3.0" (Safe-style smart-account semantics — possibly multi-sig). Treat the trust assumption as: *the address behaves like a smart account today, but the underlying EOA private-key holder can revoke or replace the delegation in a single 7702 transaction*. The chain-of-trust is therefore both the EOA key and the delegate's access logic.
9. **Beacon ownership transfer is reachable through a factory upgrade.** OZ v5 `UpgradeableBeacon` extends `Ownable`, so `transferOwnership` and `renounceOwnership` exist on the beacon. Today the factory only exposes `upgradeBeacon()` (which calls `_beacon.upgradeTo`), so the beacon owner cannot rotate under current code. But the factory is UUPS-upgradeable by its DEFAULT_ADMIN_ROLE; any one of the three factory admins (`0x9FC9…`, `0x13378…`, `0xc4233…`) can deploy a new factory implementation that calls `_beacon.transferOwnership(...)`. Beacon ownership is therefore *de facto* tied to those three keys, not architecturally immutable.
10. **Two stale repo artefacts.** `packages/contracts/deployments/42161.json` lists outdated NetworkController and SoftCap addresses. None of the broadcast files under `packages/portal-contracts/broadcast/Deploy.s.sol/42161/` produced the production portal addresses listed in the README, so there is no in-repo record of how the production portal contracts were deployed. The README is the only authoritative source for production addresses.

---

## Methodology

For each contract:

1. **Source review** of `packages/contracts/src/` and `packages/portal-contracts/src/` to identify access-control patterns (Ownable, AccessControl, custom modifiers), every privileged function, role hashes, and upgrade hooks.
2. **Proxy slot reads** (EIP-1967 admin slot `0xb53127…6103` and implementation slot `0x360894…2bbc`) for every upgradeable contract, plus the beacon's `owner()` and `implementation()` for the portal beacon.
3. **Event log enumeration** of `RoleGranted` and `RoleRevoked` for every AccessControl contract on Arbitrum One, reconciled chronologically to derive the live role membership.
4. **Spot-check `hasRole`** calls against suspect holders to confirm parsed state.
5. **Account-type verification** by calling `cast code` at every privileged address. Empty bytecode (`0x`) ⇒ bare EOA. Bytecode beginning with the EIP-7702 delegation prefix `0xef0100` ⇒ delegated EOA, with the delegate address being the trailing 20 bytes (e.g. `0xef010063c0c19a282a1b52b07dd5a65b58948a07dae32b` ⇒ delegate `0x63c0c19a282a1b52b07dd5a65b58948a07dae32b`). Any other bytecode ⇒ a deployed contract; for proxies the EIP-1967 implementation slot was read to identify the underlying logic contract.
6. **Token reads** on each chain's RPC (Ethereum, Base, BNB) for owner/minter/bridge/role accessors.

Snapshot was taken on 2026-04-30; Arbitrum block height 457,895,969. Re-run the queries (see `cast` invocations in the audit transcript) before relying on this document for security-sensitive decisions.
