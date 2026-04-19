# StoaChain Yang Emission System

This document describes the **Yang Emission** system — the mechanism that mints new STOA tokens as block rewards, split 90% to the miner and 10% to the UrStoaVault (which distributes to URSTOA stakers via an RPS model).

> **Supersedes earlier drafts.** Previous versions of this document described:
>
> - a `(CEILING − supply) / (BPD × EMISSION_SPEED × CHAINS)` emission formula;
> - a `global-supply-register` and `external-fpa` set of `chain-data` extensions;
> - a `Chainweb.Pact.GlobalSupply` Haskell module that queries per-chain `LocalSupply` tables;
> - a Foundation-Pending-Amount (FPA) delta-tracking mechanism for cross-chain settlement; and
> - a Pact-5 fork ("AncientPact") that extends `PublicData`.
>
> **None of that is in the live node.** StoaChain uses **stock upstream Pact 5.4** with no `chain-data` extensions, and emission is computed entirely inside the Pact coin module using a calendar-based formula that never queries a global-supply aggregate. The corrected narrative below is authoritative.

---

## Yang vs Yin: Miner Income Sources

StoaChain miners earn STOA from two distinct sources:

| Source | Name | Description | Newly minted? |
|--------|------|-------------|---------------|
| **Block Rewards** | **Yang Emission** | Per-block emission computed by `coin.URC_Emissions` | ✅ Yes |
| **Transaction Fees** | **Yin Earnings** | Gas fees from transactions in the block | ❌ No (transfer) |

Yang emission is newly minted; 90% goes to the miner, 10% goes to the UrStoaVault. Yin gas fees are transferred from senders; 100% goes to the miner. See [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md) for the Yin side.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MINER EARNINGS PER BLOCK                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YANG (Emission)              │  YIN (Gas Fees)                 │
│  ─────────────────────────────┼──────────────────────────────── │
│  • Newly minted STOA          │  • Existing STOA transferred    │
│  • 90% to miner               │  • 100% to miner                │
│  • 10% to UrStoaVault         │                                 │
│  • Steps down yearly          │  • Activity-based               │
│  • Pact-computed in coin mod  │                                 │
│                               │                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Emission Formula

### Interface

The authoritative emission function is `coin.URC_Emissions`. It returns two decimals:

```
[<block-emission> <urv-emission>]
```

- `<block-emission>` — the Yang reward credited to the miner account of the block that calls it. Same amount on every chain.
- `<urv-emission>` — the 10% share credited to the UrStoaVault (which lives on Chain 0 and distributes to URSTOA stakers).

### Derivation (calendar-based, no global-supply lookup)

The formula is entirely calendar-driven:

1. **Yearly allocation.** For the current year the coin module computes a yearly STOA allocation. This allocation steps down each year.
2. **Days in year.** A Gregorian-leap-aware day count gives 365 or 366.
3. **Blocks in year.** Days × `BPD` × `chains` gives the total number of blocks expected across all chains in that year (`BPD` = blocks per day = `86400 / 30 = 2880` at the 30-second delay; `chains` = 10).
4. **Per-chain per-block emission.** `yearly-allocation / blocks-in-year` is the full Yang per-block emission.
5. **90/10 split.** `block-emission` = 90% of the per-block emission; `urv-emission` = the remaining 10%.

The original derivation was recursive (defined in terms of the previous year's supply), which Pact cannot express. It was rewritten into a **linear equivalent** so Pact can evaluate it without iteration, and without needing to know the current chain supply at emission time. That is why no Pact-5 extension is required.

```pact
;; Pseudocode sketch of what the function does (see pact/stoa-coin/new-coin.pact
;; for the live source).
(defun URC_Emissions:[decimal]
    @doc "Computes the current Block Emission.
          Outputs two values:
          [<block-emission> <urv-emission>]
            <block-emission> = how much each block on each chain gets = 90% split to all chains
            <urv-emission>   = how much the UrstoaVault gets = 10% from all chains"
    ...
)
```

### Why this shape

Because the formula doesn't depend on the current per-chain supply or any cross-chain aggregate, the coin module does **not** need a `global-supply-register` injected via `chain-data`, and the node does **not** need a Haskell-side supply aggregator before coinbase. Pact 5 runs stock.

---

## 90/10 Split

Every block's Yang emission is split:

| Recipient | Share | What happens |
|-----------|-------|--------------|
| Miner | 90% (`block-emission`) | Credited to the miner account |
| UrStoaVault | 10% (`urv-emission`) | Credited to the vault (on Chain 0); distributed to stakers via RPS |

The split is handled inside the coin module as part of the coinbase path. Across 10 chains, 10 × `urv-emission` worth of STOA flows into the vault per block-epoch.

---

## URSTOA Token

**URSTOA** is a secondary token defined **inside the coin module** (not a separate module). It represents fractional ownership of the vault's virtual-mining rights.

| Property | Value |
|----------|-------|
| **Total Supply** | 1,000,000 URSTOA (fixed, minted at genesis) |
| **Precision** | 3 decimals |
| **Chain** | **Chain 0 only** |
| **Interface** | `stoa-ns.ur-stoic-fungible-v1` |
| **Purpose** | Staking in UrStoaVault to earn the 10% Yang share |

### Genesis allocation

The 1,000,000 URSTOA supply is minted at genesis on Chain 0 and allocated:

| Allocation | Amount | Purpose |
|------------|--------|---------|
| Founders | 250,000 URSTOA | Split between the blockchain founders |
| ICO sale | 250,000 URSTOA | 1 URSTOA per $5 contributed to the ICO |
| Foundation | 500,000 URSTOA | Foundation reserve |

URSTOA is live on-chain today — the explorer exposes "UrStoa Rich List" and "Vault Participation" tabs.

### Chain 0 restriction

URSTOA exists only on Chain 0. The staking logic (RPS distribution) runs on Chain 0 and there is no need to fragment staking state across 10 chains. Chain 0 is also where the UrStoaVault lives, since that is where the 10% Yang share settles every block.

---

## UrStoaVault (Staking)

The **UrStoaVault** is the sink for the `urv-emission` returned by `coin.URC_Emissions` every block. URSTOA holders stake into the vault to earn a proportional share of the 10% Yang-emission credit. Distribution uses a standard **Reward Per Share (RPS)** model so per-staker accounting is O(1).

### RPS mechanism

```
On each credit of <urv-emission> to the vault:
    RPS += urv-emission / total-staked-urstoa

On stake / unstake / collect:
    user-pending = user-stake × (RPS − user-last-RPS)
    user-last-RPS = RPS
```

This avoids iterating over stakers on every block. Stakes and rewards scale indefinitely.

### Block-emission flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Block Emission Flow                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Block mined on chain C → coin.URC_Emissions returns               │
│     [block-emission, urv-emission]                                  │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │ block-emission → miner    │    (90% of Yang — all chains)  │
│        └───────────────────────────┘                                │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │ urv-emission → UrStoaVault │   (10% of Yang — settled      │
│        │ (on Chain 0)              │    per the coin module's       │
│        └───────────────────────────┘    emission code)              │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │ RPS update — all stakers  │                                │
│        │ accrue proportionally     │                                │
│        └───────────────────────────┘                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> The UrStoaVault logic was slightly incorrect in the genesis coin module and has since been corrected via a **post-genesis module upgrade**. Fetch the live coin module via `describe-module` or the explorer for the authoritative current behavior. Do not rely on the genesis source for vault specifics.

### Granularity example

Assume 500,000 URSTOA staked and a minimum stake of 0.001 URSTOA. A 0.001 URSTOA stake represents a 2e-9 fraction of the vault. If the vault receives 0.5 STOA in a block, the minimum staker's share is 1e-9 STOA — well within STOA's 12-decimal precision.

---

## Supply Tracking

Each chain's coin module maintains a **supply table** that is updated on mint, burn, and transfer paths. The explorer reads that table directly to report live per-chain supply. There is **no Haskell-side supply register** and **no cross-chain aggregation** injected into `chain-data`: aggregation (when a caller needs a global view) is done inside Pact by reading the table on the chain being queried.

Because `coin.URC_Emissions` is calendar-driven rather than supply-driven, the supply table is a **record**, not an input to emission.

### Cross-chain transfers

Cross-chain transfers do not change global supply:

- Origin chain: balance decreases; supply table decreases.
- Target chain: balance increases; supply table increases.
- Net: zero change.

---

## Genesis STOA Supply

At genesis (2026-02-23T18:00:00 UTC), **Chain 0** receives all initial supply. On Chains 1-9 the genesis transaction is a no-op.

**STOA — 16,000,000 total**:

| Allocation | Amount | Purpose |
|------------|--------|---------|
| ICO | 10,000,000 STOA | Held in the foundation account until the ICO finalises, then distributed |
| Foundation | 2,000,000 STOA | Foundation treasury |
| Ouronet migration | 4,000,000 STOA | Migration allocation for holders from the prior Ouronet project (which ran on Kadena) |

---

## Kadena legacy systems — retained but not authoritative

> **Correction to earlier drafts.** Previous versions of this document described the CSV reward file and the allocations system as "removed". That is not an accurate description of the code shipped today.

### CSV-based miner rewards — retained, ignored

`rewards/miner_rewards.csv` (Kadena's 120-year per-block reward schedule) **is still present** in the node repo and is still read by the node. Extracting the CSV machinery cleanly from `chainweb-node` was not feasible, so the file and its plumbing were left in place.

However, the live emission **does not use that CSV value**. The coin module computes its own emission via `coin.URC_Emissions`, and the amount actually minted is the Pact-computed amount. The CSV is vestigial.

Net effect: reading from the CSV still happens, but the result is overridden. Treat the CSV as dead code surface, not as a driver of on-chain behavior.

### Allocations system

Kadena's pre-genesis allocations (CSV-based investor/team vesting) are not part of StoaChain's genesis flow. The live chain's initial supply is established by the genesis transactions defined under `pact/genesis/stoa/` and `pact/stoa-coin/new-coin.pact` in the node repo.

---

## Comparison with Kadena

| Aspect | Kadena | StoaChain |
|--------|--------|-----------|
| Emission source | CSV file (`miner_rewards.csv`) | Calendar-based formula in coin module (`coin.URC_Emissions`) |
| Authoritative value | CSV | Pact-computed |
| Split | 100% miner | 90% miner / 10% UrStoaVault |
| Secondary token | — | URSTOA (1M supply on Chain 0, stakes into UrStoaVault) |
| Supply tracking | Implicit | Per-chain Pact table in coin module |
| Pact fork required? | — | No — stock upstream Pact 5.4 |

---

## Files Reference

Paths refer to the node repo (`github.com/StoaChain/stoa-chain`).

| File | Purpose |
|------|---------|
| `pact/stoa-coin/new-coin.pact` | Genesis coin module (single file). Defines STOA, URSTOA, UrStoaVault, `URC_Emissions`, supply table, RPS distribution. |
| `pact/genesis/stoa/` | Per-chain genesis payloads (YAML/pact) for the `stoa` network. |
| `src/Chainweb/Version/Stoa.hs` | `ChainwebVersion` definition for `stoa` (genesis time, chain graph, block delay, bootstraps). |
| `src/Chainweb/Chainweb/Configuration.hs` | `_configBlockGasLimit` (1.6M default) and the upstream `_configMinGasPrice` static floor. |
| `rewards/miner_rewards.csv` | Retained for structural reasons; read but ignored (Pact value wins). |

There is **no** `src/Chainweb/Pact/GlobalSupply.hs`, no `_pdGlobalSupplyRegister` / `_pdExternalFPA` in `PublicData`, and no `Chainweb.Pact.GlobalSupply` module. Earlier docs that mentioned those files were documenting a planned design that was rendered unnecessary by the linear calendar formula.

### Note on live modules

The coin module has been **upgraded post-genesis** (primarily to fix vault logic that was slightly incorrect at genesis). The live `stoa-ns.*` module set visible on the explorer includes `stoic-predicates`, `stoic-xchain`, `stoic-fungible-v1`, `ur-stoic-fungible-v1`, `fungible-v1`, `fungible-xchain-v1`, and `gas-payer-v1`, plus upgraded `coin` and `ns` in the root namespace. For authoritative live behavior, fetch via `describe-module` or inspect the explorer — don't assume the genesis source matches current state.

---

## Repository References

- **Node repo**: https://github.com/StoaChain/stoa-chain — the Haskell chainweb-node fork that runs StoaChain.
- **Pact**: stock upstream Pact 5.4, pulled in via the `cabal.project` source-repository-package pin. No StoaChain-specific Pact fork; no `chain-data` extensions. Earlier drafts referred to a private "AncientPact" fork with extended `PublicData` fields — that never shipped and is not how the live chain works.

---

## Related Documentation

- **Gas Price System**: [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md)
- **Genesis System**: [`GENESIS_SYSTEM.md`](GENESIS_SYSTEM.md)
- **Pact 4 retrospective**: [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md)

---

*Last updated: 2026-04-19*
