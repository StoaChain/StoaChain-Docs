# StoaChain

<p align="center">
<img src="../../assets/StoaLogo.png" width="200" height="200" alt="StoaChain Logo" title="StoaChain">
</p>

<h3 align="center">A Proof-of-Work Parallel-Chain Protocol</h3>

> **StoaChain** is a **software fork** of Kadena's `chainweb-node`, running as its own independent blockchain with its own genesis (2026-02-23). It is **not a chain fork** of Kadena: it shares no ledger history with Kadena's mainnet. Live on-chain execution is Pact 5.4; the node retains Pact 4 infrastructure internally.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Built with Haskell](https://img.shields.io/badge/Built%20with-Haskell-5e5086.svg)](https://www.haskell.org/)
[![Pact Version](https://img.shields.io/badge/Pact-5.4-blue.svg)](https://pact-language.readthedocs.io/)

---

## Table of Contents

- [Overview](#overview)
- [Key Differences from Kadena](#key-differences-from-kadena)
- [Network](#network)
- [Architecture](#architecture)
- [Documentation Index](#documentation-index)
- [Building from Source](#building-from-source)
- [Running a Node](#running-a-node)
- [Block Capacity & Throughput](#block-capacity--throughput)
- [STOA Token Economics](#stoa-token-economics)
  - [Emission Formula](#emission-formula)
  - [Genesis Supply](#genesis-supply)
  - [Supply Tracking & Global Supply Registry](#supply-tracking--global-supply-registry)
  - [URSTOA Token](#urstoa-token)
  - [UrStoaVault (Staking)](#urstoavault-staking)
  - [Supply tracking](#supply-tracking)
  - [REPL Testing Notes](#repl-testing-notes)
- [Governance](#governance)
- [Pre-Launch Configuration: Keys and Addresses](#pre-launch-configuration-keys-and-addresses)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

StoaChain is a **braided, parallelized Proof-of-Work blockchain** built on the Chainweb consensus protocol. It is a minimal software fork of `chainweb-node` with the following modifications:

- **STOA Token**: A new native token, emission computed in Pact
- **Pact 5.4 on-chain**: All live on-chain execution uses Pact 5.4 (final Pact release). The node still carries Pact 4 infrastructure internally — extraction was attempted and abandoned (see [PACT4_REMOVAL.md](PACT4_REMOVAL.md))
- **Raised block gas limits**: 1.6M default / 2M max (vs upstream 150k/180k)
- **New ChainwebVersion**: A single `stoa` network (`0x0000000A`), 10 chains, Petersen graph

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
| **Native Token** | KDA (`coin` module) | STOA (live modules under `stoa-ns.*`) |
| **Token Interface** | `fungible-v2` + `fungible-xchain-v1` | `stoa-ns.stoic-fungible-v1` / `stoa-ns.fungible-xchain-v1` |
| **TRANSFER Capability** | Managed (`@managed`) | Non-managed (validations preserved) |
| **Main Namespace** | `kadena` | `stoa-ns` |
| **Pact Version (on-chain)** | Pact 4 → Pact 5 migration | Pact 5.4; node retains Pact 4 infra internally |
| **Emission Model** | CSV-based (`rewards/miner_rewards.csv`) | Computed inside Pact coin module; CSV retained in code but value ignored |
| **Initial Supply** | Complex vesting schedules | Genesis mint configured in `pact/genesis/stoa/` |
| **Networks** | mainnet01, testnet04, development, recap-development | Single network: `stoa` (`0x0000000A`) |
| **Chain Count** | 20 chains (mainnet) | 10 chains (Petersen) |
| **Block Gas Limit** | 180k max / 150k default | **2M max / 1.6M default** (production runs at 2M) |
| **Gas Price** | Static minimum (1e-8 KDA) | Static minimum (inherited from upstream). Periodic ramp is a roadmap item, not shipped |

### Fungible interfaces on the live chain

The live `stoa-ns` module set (visible on the explorer) includes `stoic-fungible-v1`, `ur-stoic-fungible-v1`, `fungible-xchain-v1`, `stoic-predicates`, `stoic-xchain`, and `gas-payer-v1`, plus upgraded `coin` and `ns` in the root namespace. These modules have diverged from the genesis source in `pact/stoa-coin/new-coin.pact` via post-genesis upgrades. When documenting live behavior, prefer the on-chain source (via `describe-module`) or the explorer over the genesis file.

#### Why Non-Managed TRANSFER?

In Kadena's coin contract, `TRANSFER` is a managed capability (`@managed`) that requires:
1. Either installing the capability in code, OR
2. Adding the capability to a key before creating the transaction

**The Problem with Managed TRANSFER:**
- You cannot sign a transaction with an "empty" key (no capabilities) and add the same key with a capability
- This forces users to use two different keys in certain scenarios
- Adds complexity without meaningful security benefit

**StoaChain's Solution:**
- The `TRANSFER` capability is **non-managed** (no `@managed` annotation)
- The capability still enforces all validations (sender guard, amount checks, precision)
- The `DEBIT` capability inside TRANSFER enforces the sender's guard
- This streamlines transaction execution without compromising security

The validations inside the TRANSFER capability are sufficient to ensure transfers execute correctly and securely.

---

## Network

StoaChain runs **one network**, named `stoa`:

| Network | Version Code | Chains | Graph Type | Block Delay |
|---------|--------------|--------|------------|-------------|
| **stoa** | `0x0000000A` (= 10) | 10 | Petersen | 30 s |

Defined in `src/Chainweb/Version/Stoa.hs`. The CLI flag is `--chainweb-version stoa`. There is no separate testnet or devnet ChainwebVersion.

### Petersen Graph (10 chains)

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
│  │  module layout visible on the explorer.                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Core Services                                │    │
│  │  src/Chainweb/                                                  │    │
│  │  ├── Pact/PactService.hs      (Pact service orchestration)      │    │
│  │  ├── Version/Stoa.hs          (network definition: `stoa`)      │    │
│  │  └── Chainweb/Configuration.hs (block gas limit: 1.6M default)  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Documentation Index

Detailed documentation is available in the following locations. Links below are to the docs repo (this repo); paths like `docs/NODE_LAUNCH_CHECKLIST.md` resolve relative to the node repo and are retained for readers working in that repo directly.

| Topic | Location | Description |
|-------|----------|-------------|
| **Node Launch Checklist** | [`NODE_LAUNCH_CHECKLIST.md`](NODE_LAUNCH_CHECKLIST.md) | Pre-launch configuration — genesis time, keysets, bootstrap |
| **Yang Emission System** | [`EMISSION_SYSTEM.md`](EMISSION_SYSTEM.md) | Emission formula, 90/10 split, supply tracking |
| **Yin Earnings (Gas)** | [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md) | Current minimum gas price and planned periodic ramp (roadmap) |
| **Genesis System** | [`GENESIS_SYSTEM.md`](GENESIS_SYSTEM.md) | Genesis payload generation, transaction order, keysets |
| **Pact 4 Removal — Retrospective** | [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md) | Why Pact 4 extraction was attempted and abandoned |

---

## Building from Source

### Prerequisites

**System Dependencies (Ubuntu/Debian):**
```bash
sudo apt-get install -y \
  ca-certificates \
  libssl-dev \
  libmpfr-dev \
  libgmp-dev \
  libsnappy-dev \
  zlib1g-dev \
  liblz4-dev \
  libbz2-dev \
  libgflags-dev \
  libzstd-dev \
  librocksdb-dev
```

**Haskell Toolchain** (per `deploy.sh` in the node repo):
- GHC 9.10.1
- Cabal 3.14.1.1

Install via [GHCup](https://www.haskell.org/ghcup/):
```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
ghcup install ghc 9.10.1
ghcup install cabal 3.14.1.1
ghcup set ghc 9.10.1
```

### Build

```bash
# Clone the repository
git clone https://github.com/StoaChain/stoa-chain.git
cd stoa-chain

# Update Cabal package index
cabal update

# Build the project
cabal build chainweb-node

# Find the binary location
cabal list-bin chainweb-node
```

The deployment scripts in the node repo (`deploy.sh`, `run-stoa.sh`, `stoa-node.service`) wrap the build + run path for production nodes.

### Generate Genesis Payloads

Before running a node, you need to generate the genesis payloads:

```bash
cd cwtools
cabal run ea
```

This generates a Haskell payload module under `src/Chainweb/BlockHeader/Genesis/` for the `stoa` network (10 chains).

---

## Running a Node

### Minimal Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Storage | 250 GB SSD | 500 GB NVMe |
| Network | Public IP | Static public IP |

### Quick Start

```bash
chainweb-node --chainweb-version=stoa
```

In production the node is wrapped by `run-stoa.sh` / `stoa-node.service` from the node repo.

### Configuration

Generate a default configuration file:
```bash
chainweb-node --print-config > config.yaml
```

Key configuration options:
```yaml
chainweb:
  chainwebVersion: stoa
  p2p:
    peer:
      hostaddress:
        hostname: your-public-ip
        port: 1789
  mining:
    enable: false  # Enable for mining nodes
```

### Health Check

Verify your node is running:
```bash
curl -sk "https://localhost:1789/chainweb/0.0/stoa/cut"
```

Or hit the public endpoints:
```bash
curl -s https://node1.stoachain.com/info
curl -s https://node2.stoachain.com/info
```

---

## Block Capacity & Throughput

StoaChain raises per-block gas capacity well beyond Kadena's. Production nodes currently run at the protocol maximum (2M gas per block) and have been tested near that limit without issue.

### Gas Limits

| Parameter | Kadena | StoaChain |
|-----------|--------|-----------|
| **Protocol Maximum** | 180,000 | **2,000,000** (`_versionMaxBlockGasLimit` in `src/Chainweb/Version/Stoa.hs`) |
| **Default Config** | 150,000 | **1,600,000** (`_configBlockGasLimit` in `src/Chainweb/Chainweb/Configuration.hs`) |

### Throughput Comparison

| Metric | Kadena Mainnet | StoaChain Mainnet |
|--------|----------------|-------------------|
| Chains | 20 | 10 |
| Gas/Block (default) | 150,000 | 1,600,000 |
| Gas/Block (max) | 180,000 | 2,000,000 |
| **Total Capacity/30s (default)** | 3,000,000 | 16,000,000 |
| **Total Capacity/30s (max)** | 3,600,000 | 20,000,000 |

Even with half the chain count, total throughput is materially higher than Kadena's.

### Configuring Gas Limit

Node operators can adjust the gas limit via:

**Config file:**
```yaml
gasLimitOfBlock: 1600000  # Up to 2000000
```

**Command line:**
```bash
chainweb-node --block-gas-limit=2000000
```

---

## STOA Token Economics

> **About the CSV** — The node code still contains `rewards/miner_rewards.csv` and still reads it. The Pact coin module computes its own emission and the minted amount is the Pact-computed amount; the CSV value is ignored. Describing the CSV as "removed" is inaccurate — it is retained for structural/compatibility reasons but is not authoritative.

### Emission Formula

STOA emission is computed entirely in the Pact coin module. The authoritative function is `coin.URC_Emissions`, which returns two values per block:

```
[<block-emission> <urv-emission>]

  <block-emission>  →  paid to the miner (90% share, same across all 10 chains)
  <urv-emission>    →  credited to the UrStoaVault (10% share, distributed via RPS)
```

The derivation is entirely calendar-based. For the current year, the coin module computes a **yearly allocation**, uses a Gregorian-leap-aware day count to get the **days-in-year**, multiplies by `BPD × chains` to get the number of blocks expected in that year, and divides the allocation across them to produce the per-chain per-block emission. The original recursive form was rewritten into a linear equivalent so Pact can evaluate it without iteration. Each year, the allocation steps down to a smaller amount.

Because the formula never needs a global-supply aggregate per block, **Pact 5 is used stock** — no custom `chain-data` fields, no Haskell-side supply register, no `Chainweb.Pact.GlobalSupply` module. The coin module does maintain its own per-chain **supply table** that is updated on mint and burn paths, so the explorer can read accurate live per-chain supply via `describe-module` / on-chain queries — that is an in-Pact data structure, not a `chain-data` extension.

### Genesis Supply

At genesis, **Chain 0** receives all initial supply.

**STOA — 16,000,000 total**:

| Allocation | Amount | Purpose |
|------------|--------|---------|
| ICO | 10,000,000 STOA | Held in the foundation account until the ICO finalises, then distributed |
| Foundation | 2,000,000 STOA | Foundation treasury |
| Ouronet migration | 4,000,000 STOA | Migration allocation for holders from the prior Ouronet project (which ran on Kadena) |

**URSTOA — 1,000,000 total** (fixed, Chain 0 only, 3-decimal precision):

| Allocation | Amount | Purpose |
|------------|--------|---------|
| Founders | 250,000 URSTOA | Split between the blockchain founders |
| ICO sale | 250,000 URSTOA | 1 URSTOA per $5 contributed to the ICO |
| Foundation | 500,000 URSTOA | Foundation reserve |

On Chains 1-9, the genesis transaction is a no-op.

### URSTOA Token

**URSTOA** is a secondary token defined **inside the coin module** (not a separate module) that acts as a perpetual virtual miner: holders who stake URSTOA in the UrStoaVault earn a proportional share of 10% of every block's Yang emission without performing any actual mining work. URSTOA is live on-chain; the explorer exposes "UrStoa Rich List" and "Vault Participation" views for it.

| Property | Value |
|----------|-------|
| **Total Supply** | 1,000,000 URSTOA (fixed, minted at genesis) |
| **Precision** | 3 decimal places (0.001 URSTOA minimum) |
| **Chain Restriction** | **Chain 0 only** |
| **Interface** | `stoa-ns.ur-stoic-fungible-v1` |
| **Purpose** | Staking in UrStoaVault to earn 10% of STOA emissions |

#### The Virtual Mining Concept

URSTOA represents **fractional ownership of perpetual mining rights**. By staking URSTOA in the UrStoaVault, holders become "virtual miners" who collectively receive 10% of every block's emissions — distributed proportionally based on their stake.

**Why 1 Million URSTOA?**
- 1M supply with 3-decimal precision allows **extremely granular** distribution of the 10% Yang share.
- Each 0.001 URSTOA represents one-billionth of the total virtual-mining power.
- Even very small stake positions can be rewarded precisely (STOA has 12 decimals).

### UrStoaVault (Staking)

The **UrStoaVault** lives on Chain 0. It is the sink for the `urv-emission` value returned by `coin.URC_Emissions` every block. URSTOA holders stake into the vault to earn a proportional share of the 10% Yang-emission credit.

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

#### RPS (Reward Per Share) Mechanism

The vault uses a standard **Reward Per Share** model for O(1) per-staker accounting:

1. When STOA is credited to the vault, `RPS += credited_amount / total_staked_urstoa`.
2. A user's pending reward = `user_stake × (current_RPS − user_last_RPS)`.
3. No loops — complexity is independent of staker count.

**Granularity example** — with 500,000 URSTOA staked, a 0.001 URSTOA minimum stake earns 0.000002% of the per-block vault credit. For a 0.5 STOA credit that is 0.00000001 STOA, well within STOA's 12-decimal precision.

> The UrStoaVault logic was slightly incorrect in the genesis coin module and has since been corrected via a post-genesis module upgrade. Fetch the live coin module via `describe-module` or the explorer for authoritative behavior.

### Supply tracking

Each chain's coin module maintains a **supply table** that the `coin.URC_Emissions` / mint / burn / transfer paths update. The explorer reads this to report live per-chain supply. There is no Haskell-side supply register and no cross-chain aggregation injected into `chain-data`: aggregation (when needed) is done inside Pact by reading the table on the chain being queried.

### Pact is stock — no `chain-data` extensions

Earlier drafts of this doc described two custom `chain-data` fields (`global-supply-register` and `external-fpa`) and a supporting `Chainweb.Pact.GlobalSupply` Haskell module. **None of that is in the live node.** StoaChain uses unmodified upstream Pact 5.4 and does not extend `chain-data`. The emission formula was rewritten to avoid needing a per-block global-supply value, which is why no Pact-level extension was necessary.

---

## Governance

### Namespace Structure

StoaChain uses the `stoa-ns` namespace (replacing Kadena's `kadena` namespace). The live module set under `stoa-ns` includes `stoic-predicates`, `stoic-xchain`, `stoic-fungible-v1`, `ur-stoic-fungible-v1`, `fungible-v1`, `fungible-xchain-v1`, and `gas-payer-v1`.

### Keyset governance

The coin module is governed by **7 Stoa Masters keysets** — `stoa-ns.stoa_master_one` through `stoa-ns.stoa_master_seven` — combined with `enforce-one`, so **any 1-of-7** master keyset can authorise a governance action on the coin module. This governs the token, emission, and URSTOA/vault code paths.

Namespace operations (creating new namespaces under `user` / `free`, administering `stoa-ns`) are gated by a separate `ns-admin-keyset` / `ns-operate-keyset` pair defined in the genesis YAMLs. These are not the coin-module governance.

A `stoa-foundation` account holds the foundation's genesis STOA allocations — including the 10M held until the ICO finalises — and is controlled by a foundation keyset.

---

## Pre-Launch Configuration: Keys and Addresses

> ⚠️ **Historical note.** Earlier drafts of this README described a centralised configuration system based on `stoachain-config.yaml` and `scripts/apply-config.sh`. **That tooling does not exist in the current node repo.** The live chain is launched via `deploy.sh`, `run-stoa.sh`, and a systemd `stoa-node.service`. Per-file edits to the genesis YAMLs under `pact/genesis/stoa/` remain necessary.

### Genesis configuration surface

Key paths in the node repo (verify before using — the genesis keyset narrative has evolved across commits):

- `src/Chainweb/Version/Stoa.hs` — ChainwebVersion definition, including `_genesisTime` (currently `2026-02-23T18:00:00.000000`) and `_versionBootstraps`.
- `src/Chainweb/Chainweb/Configuration.hs` — `_configBlockGasLimit` (1.6M default), `_configMinGasPrice` (static, inherited).
- `pact/stoa-coin/new-coin.pact` — the genesis coin contract (single module).
- `pact/genesis/stoa/keysets.yaml`, `pact/genesis/stoa/ns.yaml` — live genesis keysets.
- `pact/namespaces/ns-install.pact` — namespace module.

### Pre-launch checklist (current shape)

- [ ] Confirm `_genesisTime` in `src/Chainweb/Version/Stoa.hs` is the intended launch time.
- [ ] Configure genesis keysets in `pact/genesis/stoa/*.yaml` — the 7 Stoa Masters coin-governance keys plus the namespace admin/operate keys and the foundation keyset (see [Keyset governance](#keyset-governance)).
- [ ] `cd cwtools && cabal run ea` to regenerate genesis payloads.
- [ ] `cabal build chainweb-node` with the regenerated payloads.
- [ ] Deploy via `deploy.sh` / `stoa-node.service`.

> ⚠️ Earlier drafts of this doc asserted a Pact-vs-Haskell sync requirement between `GENESIS-TIME` in `pact/coin-contract/stoa.pact` and `stoaGenesisTime` in `src/Chainweb/GasPrice.hs`. Those files/constants do not match the current node repo layout. The live genesis time lives in `src/Chainweb/Version/Stoa.hs`.

---

## Contributing

Contributions are welcome! Please read our contributing guidelines before submitting pull requests.

### Code Style

- Follow existing Haskell conventions
- Use explicit imports
- Add documentation for new functions
- Update relevant README files

### Testing

```bash
# Run all tests
cabal test

# Run specific test suite
cabal test chainweb-tests
```

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
- **GitHub Repository**: [https://github.com/StoaChain/stoa-chain](https://github.com/StoaChain/stoa-chain)
- **Live nodes**: [https://node1.stoachain.com/info](https://node1.stoachain.com/info) · [https://node2.stoachain.com/info](https://node2.stoachain.com/info)

---

## Development Method

The extensive modifications to the Chainweb codebase—transforming it into StoaChain—were accomplished through a proprietary time dilation methodology. The StoaChain Admin, having cultivated mastery over spiritual energies—tapping into the primordial creational force that underlies existence—employed temporal manipulation capabilities to accelerate the development process.

Within a carefully constructed time dilation field, the ratio of 1 minute of external time to approximately 3 hours of internal time allowed what would normally require months of effort (learning Haskell, mastering its intricacies, understanding the complex Chainweb infrastructure) to be completed in mere hours of real-world time.

The Admin secluded himself within this temporal bubble with a laptop and a fuel-powered generator (operating at an accelerated rate to match the dilated timeframe), enabling the comprehensive overhaul of the codebase while the outside world experienced only a fraction of the elapsed duration.

---

*StoaChain - Building the future of decentralized computing*

*Last Updated: 2026-04-19*
