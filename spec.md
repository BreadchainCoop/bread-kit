# Introduction  
### Why Bread Kit?

BREAD is great. So great that many orgs have reached out to us wanting their own version. Bread Kit is our answer: a permissionless factory so anyone can deploy their own version of BREAD (basic front end included) with custom governance features, branding, and choice between many different yield distribution mechanisms. Bread Kit hands every collective the recipe for its own yield-powered currency, accelerating the flow of resources toward public goods and communities without needing to know how to code. 

This spec covers the full vision of Bread Kit. Many of these features may be far off in the future but are important to think about when designing the MVP. 


### Overview of New Features:

### Chose Yield Source 

Select between yield sources like sDAI, aave lending pools, curve USDC, etc

### New Governance Mechanisms

Select between predefined gov mechanisms like the one BREAD currently uses and governance mechanisms discovered through consumer research. Also, make it easy to swap in custom governor for deciding member projects, yield split, and updating parameters (ex. calling the setter function for epoch length). In my personal opinion, the full version of bread kit should allow for deployed controlled through the community governance, not an external admin or multi-sig. In the diagram below I still included admin but I propose that instead of admin being a multisig it uses the same voting as the Distrubtion Voting.

### Swap Between Solidarity Currencies

Currencies with the same underlying yield generation engine deployed with Bread Kit can be easily swapped with others (if a group whitelists another). 


### Pass through yield 

Instead of 100% of the yield earned being split among the member projects some of the yield can still be passed on to the user. Could the user choose their own passthrough rate or the group defines the passthrough rate.

### Cross Chain Capabilities 

Currencies aren’t locked to just one chain but main contract logic would live on a home chain. This feature is likely the most challenging.

### Subraph with Substreams

Subgraph auto-indexs all events for modules deployed through bread kit which means we can automatically provide a front end based on there customizations.

---

## Technical Implementation

```mermaid
classDiagram
%% ─── Factory ─────────────────────────────────────────
class KitFactory {
  +event KitDeployed(addr token, addr distributor, addr voting, addr admin)
  +createKit(params)
}

%% ─── Meta-Beacon & Proxies ───────────────────────────
class MetaBeacon {
  +implementation() view
  +scheduleUpgrade(newImpl)
  +executeUpgrade()
  +proposeUpgrade(newImpl)
  +acceptUpgrade()
}

class BeaconProxy { <<proxy>> }
class AdminProxy  { <<proxy>> }

KitFactory --* BeaconProxy
BeaconProxy ..> MetaBeacon
BeaconProxy ..> YieldToken
BeaconProxy ..> YieldDistributor
BeaconProxy ..> DistributionVoting
BeaconProxy ..> AdminProxy
AdminProxy  ..> MetaBeacon
AdminProxy  ..> IAdmin

%% ─── Yield Token ─────────────────────────────────────
class YieldToken {
  +mint(receiver,amt?) payable
  +burn(amt,receiver)
  +yieldAccrued() view
  +claimYield(amt,receiver)
  +swap(otherToken,amt)
  +passthroughBps
}
YieldToken ..> ERC20VotesUpgradeable
YieldToken --> IYieldAdapter
YieldToken --> SwapRegistry

%% ─── Yield Adapters (examples) ───────────────────────
class IYieldAdapter { <<interface>> }
class SDAIAdapter
class AaveAdapter
IYieldAdapter <|.. SDAIAdapter
IYieldAdapter <|.. AaveAdapter

%% ─── Swap Registry ───────────────────────────────────
class SwapRegistry {
  +whitelist(addr)
  +isWhitelisted(addr) view
}

%% ─── Admin (Governor / Multisig) ─────────────────────
class IAdmin {
  <<interface>>
  +propose(targets,data,desc)
  +vote(id,choice)
  +execute(id)
}
class TokenMultisig
IAdmin <|.. TokenMultisig
AdminProxy --> VotingBoosters
AdminProxy --> BoostToken
AdminProxy --> YieldDistributor

%% ─── Distribution Voting (points) ────────────────────
class DistributionVoting {
  +castVote(points[])
  +castVoteWithBoost(points[],idx[])
  +currentVotes
  +projectDistributions
}
DistributionVoting --> VotingBoosters
DistributionVoting --> BoostToken
DistributionVoting --> YieldDistributor

%% ─── Distributor (stores & splits) ───────────────────
class YieldDistributor {
  +resolveDistribution() view
  +distributeYield()
  +setPassthrough(bps)
  +setCycleLength(blocks)
  +setFixedSplitDivisor(div)
  +setMinVotingPower(pwr)
  +setMaxPoints(pts)
  +queueProjectAddition(addr)
  +queueProjectRemoval(addr)
}
YieldDistributor --> YieldToken
YieldDistributor --> BoostToken

%% ─── Boosters / Multipliers ──────────────────────────
class VotingBoosters {
  +addBooster(IBooster)
  +removeBooster(IBooster)
  +getTotalBoost(user,idx[]) view
}
class IBooster { <<interface>> }
VotingBoosters ..> IBooster

%% ─── Optional Boost Token ────────────────────────────
class BoostToken { <<ERC20Votes>> }

```

### Module Snapshot

| Contract / Module                          | Generic Roles & Notes                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **YieldToken**                             | Wraps the collateral asset (e.g., wxDAI) into a yield-bearing wrapper (e.g., sDAI) and tokenises it. `yieldAccrued()` = wrapper assets – totalSupply; `claimYield()` mints new YieldToken so principal never leaves the adapter.                                                                                                                                                          |
| **DistributionVoting**                     | Collects point-based votes (`castVote`, `castVoteWithBoost`) that decide the **relative split among member projects**. Uses balances of **YieldToken + BoostToken** and any weight multipliers in **VotingBoosters**. After each vote it writes the normalised weights to `YieldDistributor.projectDistributions`.                                                                        |
| **YieldDistributor**                       | Stores all configurable parameters and performs the actual yield split.<br>• Pays depositors their `passthroughBps` share.<br>• Pays each project its fixed-plus-voted share using the weights supplied by `DistributionVoting`.<br>• Setter functions (`setCycleLength`, `setPassthrough`, etc.) are guarded by **`onlyAdmin`** so they can be changed through the kit’s Admin/Governor. |
| **Admin / Governor (IAdmin + AdminProxy)** | Pluggable module (token-weighted multisig by default) that governs parameter changes and upgrades. Exposes `propose / vote / execute`; on execution it can call any target, typically `YieldDistributor` setters or MetaBeacon upgrade functions. Kits may swap this for their own governor template.                                                                                     |
| **VotingBoosters**                         | Registry of off-chain or on-chain multiplier contracts (NFT badges, referral tokens, liquidity staking) that can increase a holder’s voting weight in **DistributionVoting** (and, if desired, in the Admin governor).                                                                                                                                                                    |
| **BoostToken**                             | Optional secondary ERC20Votes token (e.g., Buttered Bread) whose balance is counted by boosters and/or governor templates.                                                                                                                                                                                                                                                                |
| **KitFactory**                             | Deploys a trio *(YieldToken, YieldDistributor, DistributionVoting)* behind BeaconProxies (plus an AdminProxy). Initializes them with chosen adapter, initial parameters, and chosen governor template.                                                                                                                                                                                    |
| **SwapRegistry**                           | Shared contract that whitelists compatible YieldToken contracts so holders can 1-to-1 swap solidarity currencies (same adapter).                                                                                                                                                                                                                                                          |
| **MetaBeacon**                             | Holds the logic address for each contract type and supports three upgrade modes:<br>1. **Core-timelock auto-upgrade** (default).<br>2. **Broadcast + per-kit opt-in** via `acceptUpgrade()`.<br>3. **Private beacon** a kit can switch to for full autonomy.                                                                                                                              |
| **BeaconProxy**                            | Lightweight proxy deployed per kit that reads its implementation address from a (meta or private) beacon on every call.                                                                                                                                                                                                                                                                   |
| **UpgradeGovernor (optional)**             | Alternate governor template a kit can select at deploy time (e.g., QuadraticGovernor). Uses the same `IAdmin` interface, so it can replace the default multisig without contract migrations.                                                                                                                                                                                              |




---

## User Flows

### 1. Minting YieldToken

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI
    participant YieldToken
    participant CollateralWrapper
    participant YieldWrapper

    alt Pay with native collateral
        User->>YieldToken: mint{value: X} (receiver=User)
        YieldToken->>CollateralWrapper: deposit{value: X}
    else Pay with wrapped collateral
        User->>CollateralWrapper: approve(YieldToken, X)
        UI->>YieldToken: mint(receiver, X)
    end
    YieldToken->>YieldWrapper: deposit(X, YieldToken)
    YieldToken-->>User: ERC20 shares minted
```

### 2. Voting & Boosts

```mermaid
sequenceDiagram
    autonumber
    actor Holder
    participant UI
    participant DistributionVoting
    participant VotingBoosters

    Holder->>UI: open vote modal
    UI->>VotingBoosters: getValidBoosterIdx(Holder)
    Holder->>UI: assign points per project
    UI->>DistributionVoting: castVoteWithBoost(points[], idx[])
```

### 3. Automated Distribution

```mermaid
sequenceDiagram
    autonumber
    participant Keeper
    participant YieldDistributor
    participant YieldToken
    participant Projects

    Keeper->>YieldDistributor: resolveDistribution()
    alt ready
        Keeper->>YieldDistributor: distributeYield()
        YieldDistributor->>YieldToken: claimYield(yield)
        YieldDistributor->>Projects: transfer fixed & voted splits
    else not yet
        Note over Keeper: try again later
    end
```
---

## MVP

Thought for a couple of seconds


## MVP Scope (v0.1 “Bake-Starter”)

| ‎                       | Included in MVP                                                                                                                                                                              | Deferred (post-MVP)                                                  |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Yield sources**       | *Two adapters only*: **sDAI** (Gnosis) and **aUSDC** (Aave v3 on Polygon or Mainnet).                                                                                                        | Any other adapters (Curve, Pendle, LSTs).                            |
| **Admin (Address / Multisig)** | A single address passed into **`KitFactory.createKit()`** (`params.admin`).<br>• Can be an EOA for small collectives **or** any multisig / governor the deployer already trusts.<br>• Must expose the simple *gnosis-safe-style* `exec(target,data)` call the factory expects. (No on-chain proposal/vote machinery is included in MVP.)<br>• Holds `ONLY_ADMIN` rights on `YieldDistributor` setters and on beacon–upgrade opt-out.<br>• If the deployer later wants community governance, they can migrate control by having the current admin call `setAdmin(newAddr)` once that function is added in a post-MVP upgrade. |
| **Distribution voting** | Single contract `DistributionVoting` (points + boosters).                                                                                                                                    | Referral boosters, reputation curves, external weighting oracles.    |
| **YieldDistributor**    | Pays: depositor pass-through % + fixed divisor + voted share. Setters gated by `onlyAdmin`.                                                                                                  | Streaming payouts, protocol fee skims, emergency withdraw.           |
| **YieldToken**          | ERC20Votes wrapper + *sDAI* and *aUSDC* adapters. Passthrough basis-points fixed at deploy (can change via Admin).                                                                           | SwapRegistry, token-to-token swaps, cross-chain mint/burn, gasless.  |
| **Upgrade infra**       | **BeaconProxy** per kit, implementation address set by BreadChain Co-op timelock. Kits may “divorce” to private beacon, but **no meta-beacon** tree yet.                                     | Meta-beacon, broadcast upgrades, DAO-controlled beacons.             |
| **Other exclusions**    | No SwapRegistry, no MetaTx relayer, no BridgeAdapter, no auto-subgraph/ substreams (hand-rolled subgraph only).                                                                              |                                                                      |

---

## MVP Contract Diagram

```mermaid
classDiagram
%% ── Factory ───────────────────────────────────
class KitFactory {
  +event KitDeployed(addr token, addr distributor, addr voting, addr admin)
  +createKit(params)
}
KitFactory --* BeaconProxy

%% ── Beacon proxy (single level) ───────────────
class BeaconProxy { <<proxy>> }
BeaconProxy ..> YieldToken
BeaconProxy ..> YieldDistributor
BeaconProxy ..> DistributionVoting
BeaconProxy ..> AdminMultisig

%% ── Yield Token ───────────────────────────────
class YieldToken {
  +mint(receiver,amt?) payable
  +burn(amt,receiver)
  +yieldAccrued() view
  +claimYield(amt,receiver)
  +passthroughBps
}
YieldToken ..> ERC20VotesUpgradeable
YieldToken --> SDAIAdapter
YieldToken --> AaveUSDCAdapter

class SDAIAdapter
class AaveUSDCAdapter

%% ── Admin multisig ────────────
class AdminMultisig {
  +user logic

}
AdminMultisig --> YieldDistributor

%% ── Distribution voting (points) ───────────────
class DistributionVoting {
  +castVote(points[])
  +castVoteWithBoost(points[],idx[])
  +currentVotes
  +projectDistributions
}
DistributionVoting --> VotingBoosters
DistributionVoting --> YieldDistributor

%% ── Booster registry (optional) ────────────────
class VotingBoosters {
  +addBooster(IBooster)
  +getTotalBoost(user,idx[]) view
}
class IBooster { <<interface>> }
VotingBoosters ..> IBooster

%% ── Yield distributor (splitter) ───────────────
class YieldDistributor {
  +resolveDistribution() view
  +distributeYield()
  +setPassthrough(bps)   <<onlyAdmin>>
  +setCycleLength(blocks)<<onlyAdmin>>
  +setFixedSplitDivisor(div)<<onlyAdmin>>
  +queueProjectAddition(addr)<<onlyAdmin>>
  +queueProjectRemoval(addr)<<onlyAdmin>>
}
YieldDistributor --> YieldToken
YieldDistributor --> BoostToken

%% ── Optional boost token ───────────────────────
class BoostToken { <<ERC20Votes>> }
```

**Key notes**

* **One beacon per contract type**, owned by a BreadChain timelock (48 h).
  *Setter changes* and *adapter switches* require an on-chain proposal in `AdminMultisig`.
* **DistributionVoting** pushes normalised weights to `YieldDistributor.projectDistributions`.
  `distributeYield()` trusts that array and performs transfers atomically.
* **Adapters hard-whitelisted** in `KitFactory`; no dynamic adapter registry in MVP.

This trimmed-down surface minimises audit scope yet delivers:

* Community-controlled point voting for project funding.
* Choice of two stablecoin yield engines day-1.
* Pass-through yield, fixed-plus-voted split, and upgrade path via timelocked beacon.



---
