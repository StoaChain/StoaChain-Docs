# StoaChain

<p align="center">
<img src="assets/StoaLogo.png" width="200" height="200" alt="StoaChain Logo" title="StoaChain">
</p>

<h3 align="center">A Proof-of-Work Parallel-Chain Protocol</h3>

> **StoaChain** is a blockchain built on Kadena's Chainweb protocol. It is a **software fork** of `chainweb-node` — a minimal set of modifications over upstream, running as its own independent blockchain with its own genesis (2026-02-23). It is **not a chain fork** of Kadena: it shares no ledger history with Kadena's mainnet.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Built with Haskell](https://img.shields.io/badge/Built%20with-Haskell-5e5086.svg)](https://www.haskell.org/)
[![Pact Version](https://img.shields.io/badge/Pact-5.4-blue.svg)](https://pact-language.readthedocs.io/)

---

## Table of Contents

- [Overview](#overview)
- [Key Differences from Kadena](#key-differences-from-kadena)
- [Networks](#networks)
- [Architecture](#architecture)
- [Documentation Index](#documentation-index)
- [STOA Token Economics](#stoa-token-economics)
  - [Emission Formula](#emission-formula)
  - [URSTOA Token](#urstoa-token)
  - [UrStoaVault (Staking)](#urstoavault-staking)
  - [Chain-Data Extensions](#chain-data-extensions)
- [Governance](#governance)
- [Pre-Launch Configuration](#pre-launch-configuration)
- [Source Repositories](#source-repositories)
- [License](#license)
- [Development Method](#development-method)

---

## Overview

StoaChain is a **braided, parallelized Proof-of-Work blockchain** built on the Chainweb consensus protocol. It inherits Kadena's multi-chain architecture with the following modifications:

- **STOA Token**: A new native token with deterministic emission computed in Pact
- **Pact 5.4 on-chain**: All live on-chain execution uses Pact 5.4 (the final Pact release)
- **Raised gas limits**: 1.6M default / 2M max block gas (vs upstream 150k/180k)
- **New ChainwebVersion**: A single `stoa` network with its own genesis

> **Note on codebase state** — The node repo still carries Pact 4 infrastructure internally (extraction was attempted and abandoned because Pact 4 and Pact 5 share code paths). The *live chain* runs Pact 5.4 regardless. See [docs/chainweb-node/PACT4_REMOVAL.md](docs/chainweb-node/PACT4_REMOVAL.md) for the retrospective.

### What is Chainweb?

Chainweb is a braided multi-chain architecture that:
- Runs **multiple parallel chains** that reference each other's blocks
- Achieves **horizontal scalability** without sacrificing security
- Maintains **Bitcoin-level security** through Proof-of-Work consensus

Read the original whitepapers:
- [Chainweb: A Proof-of-Work Parallel-Chain Architecture](https://d31d887a-c1e0-47c2-aa51-c69f9f998b07.filesusr.com/ugd/86a16f_029c9991469e4565a7c334dd716345f4.pdf)
- [Agent-based Simulations of Blockchain Protocols](https://d31d887a-c1e0-47c2-aa51-c69f9f998b07.filesusr.com/ugd/86a16f_3b2d0c58179d4edd9df6df4d55d61dda.pdf)

---

## Key Differences from Kadena

| Aspect | Kadena Chainweb | StoaChain |
|--------|-----------------|-----------|
| **Native Token** | KDA (`coin` module) | STOA (`stoa-ns.*` modules, see governance note) |
| **Token Interface** | `fungible-v2` + `fungible-xchain-v1` | `stoa-ns.stoic-fungible-v1` / `stoa-ns.fungible-xchain-v1` |
| **TRANSFER Capability** | Managed (`@managed`) | Non-managed (simplified, validations preserved) |
| **Main Namespace** | `kadena` | `stoa-ns` |
| **Pact Version (on-chain)** | Pact 4 → Pact 5 migration | Pact 5.4 (node retains Pact 4 infrastructure internally) |
| **Emission Model** | CSV-based (`rewards/miner_rewards.csv`) | Computed inside Pact coin module; CSV retained in code but its value is ignored |
| **Initial Supply** | Complex vesting schedules | Genesis mint (final amounts configured pre-launch in `pact/genesis/stoa/`) |
| **Networks** | mainnet01, testnet04, development, recap-development | Single network: `stoa` |
| **Chain Count** | 20 chains (mainnet) | 10 chains (Petersen graph) |
| **Block Gas Limit** | 180k max / 150k default | **2M max / 1.6M default** (production nodes run at 2M) |
| **Gas Price** | Static minimum (1e-8 KDA) | Static minimum (inherited from upstream). Periodic ramp is a planned roadmap item, not shipped |

### Fungible interfaces on the live chain

The live on-chain module layout (visible on the explorer) uses `stoa-ns`-namespaced interfaces — notably `stoa-ns.stoic-fungible-v1` and `stoa-ns.fungible-xchain-v1` (plus `stoa-ns.ur-stoic-fungible-v1`, `stoa-ns.stoic-predicates`, `stoa-ns.stoic-xchain`, `stoa-ns.fungible-v1`, and `stoa-ns.gas-payer-v1`). These live modules have **diverged from the genesis source** in `pact/stoa-coin/new-coin.pact` via post-genesis upgrades, including bug-fix changes. The authoritative source for live behavior is the chain itself — fetch via `describe-module` from a node, or inspect the explorer.

**Why Non-Managed TRANSFER?**
- You cannot sign a transaction with an "empty" key (no capabilities) and add the same key with a capability
- This forces users to use two different keys in certain scenarios
- The TRANSFER capability still enforces all validations (sender guard, amount checks, precision)
- The DEBIT capability inside TRANSFER enforces the sender's guard
- This streamlines transaction execution without compromising security

---

## Network

StoaChain runs **one network**, named `stoa`:

| Network | Version Code | Chains | Graph Type | Block Delay |
|---------|--------------|--------|------------|-------------|
| **stoa** | `0x0000000A` (= 10) | 10 | Petersen | 30 s |

Defined in `src/Chainweb/Version/Stoa.hs`. The CLI flag is `--chainweb-version stoa`. There is no separate testnet or devnet ChainwebVersion — development and testing happen against the same `stoa` network.

### Chain Graph — Petersen (10 chains)

```
    0 --- 5
   /|\   /|\
  1 | \ / | 6
  |\ |  X  | /|
  | \|/ \ \|/ |
  2--+----+--7
  |/ |\  /|\ |
  | /|  X  | \|
  3 | / \ | 8
   \|/   \|/
    4 --- 9
```

Each chain has exactly 3 neighbours (degree-3 regular graph).

### Public nodes

Mainnet is currently served by two operator nodes over HTTPS (reverse-proxied on port 443 in front of the chainweb service API):

| URL | Role |
|-----|------|
| `https://node1.stoachain.com` | Canonical **bootstrap** peer (listed in `_versionBootstraps` in `src/Chainweb/Version/Stoa.hs` as `node1.stoachain.com:1789`) |
| `https://node2.stoachain.com` | Additional public endpoint. Not currently wired in as a protocol-level bootstrap peer |

Liveness check:

```bash
curl -s https://node1.stoachain.com/info
# {"nodeVersion":"stoa","nodeNumberOfChains":10,"nodeBlockDelay":30000000,
#  "nodePackageVersion":"2.32.0", ...}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          StoaChain Node                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  Pact 5.4 Execution Layer                       │    │
│  │  src/Chainweb/Pact5/                                            │    │
│  │  ├── TransactionExec.hs    (transaction execution)              │    │
│  │  ├── Templates.hs          (tx templates)                       │    │
│  │  ├── SPV.hs                (cross-chain verification)           │    │
│  │  └── Backend/ChainwebPactDb.hs (database layer)                 │    │
│  │                                                                 │    │
│  │  Pact 4 infrastructure (src/Chainweb/Pact4/) is still present   │    │
│  │  in the node code but is not exercised by the live chain.       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │          Genesis Coin Contract (single module)                  │    │
│  │  pact/stoa-coin/new-coin.pact                                   │    │
│  │                                                                 │    │
│  │  Post-genesis upgrades have introduced the live `stoa-ns.*`     │    │
│  │  module layout visible on the explorer (stoic-fungible-v1,      │    │
│  │  stoic-xchain, ur-stoic-fungible-v1, etc.).                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Core Services                                │    │
│  │  src/Chainweb/                                                  │    │
│  │  ├── Pact/PactService.hs  (Pact service orchestration)          │    │
│  │  ├── Version/Stoa.hs      (network definition: `stoa`)          │    │
│  │  └── Chainweb/Configuration.hs (block gas limit: 1.6M default)  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Documentation Index

### Chainweb Node (StoaChain)

| Document | Description |
|----------|-------------|
| [**Full Technical README**](docs/chainweb-node/README.md) | Complete technical documentation with build instructions |
| [**Node Launch Checklist**](docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md) | Pre-launch configuration guide — genesis time, keysets, bootstrap |
| [**Yang Emission System**](docs/chainweb-node/EMISSION_SYSTEM.md) | Emission formula computed in Pact (CSV retained but ignored) |
| [**Yin Earnings (Gas)**](docs/chainweb-node/GAS_PRICE_SYSTEM.md) | Minimum gas price — current state and planned periodic ramp (roadmap) |
| [**Genesis System**](docs/chainweb-node/GENESIS_SYSTEM.md) | Genesis payload generation, transaction order, keysets |
| [**Pact 4 Removal — Retrospective**](docs/chainweb-node/PACT4_REMOVAL.md) | Why Pact 4 extraction was attempted and abandoned |

### Pact 5.4

| Document | Description |
|----------|-------------|
| [**Main README**](docs/pact-5/README.md) | Overview — StoaChain runs stock upstream Pact 5.4, no fork, no `chain-data` extensions |
| [**chain-data reference**](docs/pact-5/chain-data.md) | Reference for the (unmodified) `chain-data` native as used on StoaChain |

---

## STOA Token Economics

### Emission — computed entirely in Pact

STOA emission is computed inside the Pact coin module by the function `coin.URC_Emissions`, which returns two values per block:

```
[<block-emission> <urv-emission>]

  <block-emission>  →  paid to the miner (90% share, same across all chains)
  <urv-emission>    →  credited to the UrStoaVault (10% share, distributes to URSTOA stakers)
```

The formula derives a **yearly allocation** from a Gregorian-leap-aware day count, then splits it across the blocks expected in that year (`BPD × days-in-year × chains`) to produce the per-block per-chain amount. The original recursive form was rewritten into a linear equivalent so Pact can evaluate it without iteration. Each year rolls over to a smaller allocation.

Because the formula needs only calendar arithmetic and never queries a global-supply aggregate per block, **Pact 5 is used stock** — no chain-data extensions, no Haskell-side supply register. Per-chain supply is still tracked inside the coin module via a dedicated table that mint/burn paths update; the explorer reads that table to display live per-chain supply.

> **About the CSV** — The node code still contains `rewards/miner_rewards.csv` (retained because extracting the CSV machinery cleanly was not feasible). **It is vestigial.** The value the node would otherwise serve from it is ignored; the amount actually minted is the Pact-computed amount. "CSV was removed" is not an accurate description of the code — "CSV is kept but not authoritative" is.

### Block Emission Split

| Recipient | Share | Description |
|-----------|-------|-------------|
| **Miner** | 90% | Direct block reward (Yang Emission `block-emission`) |
| **UrStoaVault** | 10% | Credited per block (`urv-emission`); distributed to URSTOA stakers via RPS |

### Miner Income Sources

| Source | Name | Description |
|--------|------|-------------|
| Block Rewards | **Yang Emission** | Newly minted STOA (90% to miner, 10% to vault) |
| Transaction Fees | **Yin Earnings** | Gas fees transferred to miner (100%) |

### Genesis Supply

At genesis, **Chain 0** receives all initial supply.

**STOA: 16,000,000 total**, split across genesis accounts as:

| Allocation | Amount | Purpose |
|------------|--------|---------|
| ICO | 10,000,000 STOA | Held in the foundation account until the ICO finalises, then distributed |
| Foundation | 2,000,000 STOA | Foundation treasury |
| Ouronet migration | 4,000,000 STOA | Migration allocation for holders from the prior Ouronet project (which ran on Kadena) |

**URSTOA: 1,000,000 total** (fixed, minted at genesis on Chain 0, 3 decimal precision):

| Allocation | Amount | Purpose |
|------------|--------|---------|
| Founders | 250,000 URSTOA | Split between the blockchain founders |
| ICO sale | 250,000 URSTOA | 1 URSTOA per $5 contributed to the ICO |
| Foundation | 500,000 URSTOA | Foundation reserve |

On Chains 1-9, the genesis transaction is a no-op (all initial supply is minted on Chain 0).

### URSTOA Token

**URSTOA** is a secondary token defined inside the coin module (not a separate module) that acts as a **perpetual virtual miner**: holders who stake URSTOA in the UrStoaVault earn a proportional share of 10% of every block's Yang emission without doing any actual mining work. URSTOA is live on-chain today — the explorer exposes a "UrStoa Rich List" and a "Vault Participation" tab.

| Property | Value |
|----------|-------|
| **Total Supply** | 1,000,000 URSTOA (fixed, minted at genesis) |
| **Precision** | 3 decimal places (0.001 URSTOA minimum) |
| **Chain Restriction** | **Chain 0 only** |
| **Interface** | `stoa-ns.ur-stoic-fungible-v1` |
| **Purpose** | Staking to earn 10% of STOA emissions |

#### The Virtual Mining Concept

URSTOA represents **fractional ownership of perpetual mining rights**. By staking URSTOA in the UrStoaVault, holders become "virtual miners" who collectively receive 10% of every block's emissions — distributed proportionally based on their stake.

### UrStoaVault (Staking)

The **UrStoaVault** lives on Chain 0 and is the sink for the `urv-emission` returned by `coin.URC_Emissions` each block. URSTOA holders stake into the vault to earn a proportional share of the 10% Yang-emission credit. Distribution uses a standard **Reward Per Share (RPS)** model for O(1) per-staker accounting:

1. When STOA is credited to the vault, `RPS += credited_amount / total_staked_urstoa`.
2. A user's pending reward = `user_stake × (current_RPS − user_last_RPS)`.
3. No loops — complexity is independent of staker count.

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

> The UrStoaVault logic was slightly incorrect in the genesis coin module and has since been corrected via a post-genesis module upgrade. The live behavior is authoritative — fetch the coin module via `describe-module` or the explorer if you need to inspect the corrected logic.

---

## Governance

### Namespace Structure

StoaChain uses the `stoa-ns` namespace (replacing Kadena's `kadena` namespace). On-chain, the live `stoa-ns` module set is visible on the explorer and includes modules such as `stoic-predicates`, `stoic-xchain`, `stoic-fungible-v1`, `ur-stoic-fungible-v1`, `fungible-v1`, `fungible-xchain-v1`, and `gas-payer-v1`.

### Keyset governance

The coin module is controlled by **7 Stoa Masters keysets** — `stoa-ns.stoa_master_one` through `stoa-ns.stoa_master_seven` — combined with `enforce-one`, so any 1-of-7 master keyset can authorise governance actions on the coin module.

Separately, the `stoa-ns`, `user`, and `free` namespaces are administered via an `ns-admin-keyset` / `ns-operate-keyset` pair defined in the genesis YAMLs. Those keysets gate namespace operations; they are not the coin-module governance.

A `stoa-foundation` account holds the foundation's genesis STOA allocations (including the 10M held until the ICO finalises) and is controlled by a foundation keyset.

---

## Pre-Launch Configuration

> ⚠️ **Historical note.** Earlier versions of this README documented a centralised-configuration system based on `stoachain-config.yaml` and `scripts/apply-config.sh`. **That tooling does not exist in the current node repo.** The live chain is deployed with `deploy.sh`, `run-stoa.sh`, and a `stoa-node.service` systemd unit. The per-file configuration points (genesis time, keysets, etc.) are still documented in [`docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md`](docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md), but must be edited directly — there is no single-file apply-all script.

Generate genesis payloads and build:

```bash
cd cwtools && cabal run ea
cd .. && cabal build chainweb-node
```

---

## Source Repository

The StoaChain node source lives at [`github.com/StoaChain/stoa-chain`](https://github.com/StoaChain/stoa-chain.git) (referenced from `deploy.sh`). It is the repo that builds the binary running on `node1.stoachain.com` and `node2.stoachain.com`.

**Pact**: StoaChain uses stock upstream Pact 5.4 — no fork, no `chain-data` extensions. The Pact source-repository-package is pinned in `cabal.project`. Earlier drafts of these docs referred to a StoaChain-specific "AncientPact" fork with extended `chain-data` fields (`global-supply-register`, `external-fpa`); those drafts are wrong and are being corrected.

---

## License

StoaChain is released under the **MIT License**.

This project is a fork of [Kadena Chainweb](https://github.com/kadena-io/chainweb-node), which is also MIT licensed.

---

## Acknowledgments

- **Kadena Team**: For creating the Chainweb protocol and Pact language
- **StoaChain Contributors**: For adapting and improving the codebase

---

## Links

- **GitBook Documentation**: [https://demiourgos-holdings-tm.gitbook.io/kadena-evolution](https://demiourgos-holdings-tm.gitbook.io/kadena-evolution)
- **Public Documentation**: [https://github.com/StoaChain/StoaChain-Docs](https://github.com/StoaChain/StoaChain-Docs)

---

## Development Method

The extensive modifications to the Chainweb codebase—transforming it into StoaChain—were accomplished through a proprietary time dilation methodology. The StoaChain Admin, having cultivated mastery over spiritual energies—tapping into the primordial creational force that underlies existence—employed temporal manipulation capabilities to accelerate the development process.

Within a carefully constructed time dilation field, the ratio of 1 minute of external time to approximately 3 hours of internal time allowed what would normally require months of effort (learning Haskell, mastering its intricacies, understanding the complex Chainweb infrastructure) to be completed in mere hours of real-world time.

The Admin secluded himself within this temporal bubble with a laptop and a fuel-powered generator (operating at an accelerated rate to match the dilated timeframe), enabling the comprehensive overhaul of the codebase while the outside world experienced only a fraction of the elapsed duration.

---

*StoaChain - Building the future of decentralized computing*

*Last Updated: 2026-04-19*
