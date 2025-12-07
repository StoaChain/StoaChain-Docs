# StoaChain

<p align="center">
<img src="assets/StoaLogo.png" width="200" height="200" alt="StoaChain Logo" title="StoaChain">
</p>

<h3 align="center">A Proof-of-Work Parallel-Chain Protocol</h3>

> **StoaChain** is a next-generation blockchain forked from Kadena's Chainweb protocol, featuring a custom economic model, streamlined architecture, and Pact 5 from genesis.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Built with Haskell](https://img.shields.io/badge/Built%20with-Haskell-5e5086.svg)](https://www.haskell.org/)
[![Pact Version](https://img.shields.io/badge/Pact-5.0-blue.svg)](https://pact-language.readthedocs.io/)

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
  - [URSTOA-Vault (Staking)](#urstoa-vault-staking)
  - [Chain-Data Extensions](#chain-data-extensions)
- [Governance](#governance)
- [Pre-Launch Configuration](#pre-launch-configuration)
- [Source Repositories](#source-repositories)
- [License](#license)
- [Development Method](#development-method)

---

## Overview

StoaChain is a **braided, parallelized Proof-of-Work blockchain** built on the Chainweb consensus protocol. It inherits Kadena's innovative multi-chain architecture while introducing significant improvements:

- **STOA Token**: A new native token with deterministic emission mechanics
- **Pact 5 Exclusive**: Uses only the latest Pact smart contract language from genesis
- **Simplified Governance**: 7 Stoa Masters keysets control module upgrades
- **No Allocations**: Dynamic emission replaces CSV-based vesting schedules
- **Streamlined Codebase**: Removed legacy Pact 4 execution code (~3,300 lines)

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
| **Native Token** | KDA (`coin` module) | STOA (`STOA` module) |
| **Token Interface** | `fungible-v2` + `fungible-xchain-v1` | `StoaFungibleV1` (merged) |
| **TRANSFER Capability** | Managed (`@managed`) | Non-managed (simplified) |
| **Governance** | `false` (always true) | 7 Stoa Masters keysets |
| **Main Namespace** | `kadena` | `stoa-ns` |
| **Pact Version** | Pact 4 → Pact 5 migration | Pact 5 from genesis |
| **Emission Model** | CSV-based allocations + miner rewards | Deterministic formula |
| **Initial Supply** | Complex vesting schedules | Single genesis mint |
| **Networks** | mainnet01, testnet04, development | stoamainnet01, stoatestnet02, stoadevnet03 |
| **Chain Count** | 20 chains (mainnet) | 10 chains (mainnet), 3 chains (testnet/devnet) |
| **Block Gas Limit** | 180k max, 150k default | 500k max, 400k default |
| **Gas Price** | Static minimum (1e-8 KDA) | Dynamic minimum (time-based) |

### StoaFungibleV1 Interface

StoaChain uses a simplified `StoaFungibleV1` interface that merges `fungible-v2` and `fungible-xchain-v1` into a single interface. A key simplification is the **removal of managed capabilities** from the TRANSFER function.

**Why Non-Managed TRANSFER?**
- You cannot sign a transaction with an "empty" key (no capabilities) and add the same key with a capability
- This forces users to use two different keys in certain scenarios
- The TRANSFER capability still enforces all validations (sender guard, amount checks, precision)
- The DEBIT capability inside TRANSFER enforces the sender's guard
- This streamlines transaction execution without compromising security

---

## Networks

StoaChain supports **three networks**:

| Network | Version Code | Chains | Graph Type | Purpose |
|---------|--------------|--------|------------|---------|
| **StoaMainnet01** | `0x00000015` | 10 | Petersen | Production network |
| **StoaTestnet02** | `0x00000016` | 3 | Triangle | Testing network |
| **StoaDevnet03** | `0x00000017` | 3 | Triangle | Development (PoW disabled) |

### Chain Graph Configurations

**Petersen Graph (10 chains - Mainnet)**
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

**Triangle Graph (3 chains - Testnet/Devnet)**
```
    0
   / \
  1---2
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          StoaChain Node                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Pact 5 Execution Layer                       │    │
│  │  src/Chainweb/Pact5/                                            │    │
│  │  ├── TransactionExec.hs    (transaction execution)              │    │
│  │  ├── Templates.hs          (STOA tx templates)                  │    │
│  │  ├── SPV.hs                (cross-chain verification)           │    │
│  │  └── Backend/ChainwebPactDb.hs (database layer)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    STOA Token Contract                          │    │
│  │  pact/coin-contract/stoa.pact                                   │    │
│  │  ├── STOA Submodule                                             │    │
│  │  │   ├── XM_StoaCoinbase      (block rewards)                   │    │
│  │  │   ├── C_Transfer           (transfers)                       │    │
│  │  │   └── C_TransferAcross     (cross-chain)                     │    │
│  │  ├── URSTOA Submodule (Chain 0 only)                            │    │
│  │  │   └── C_UR|Transfer        (URSTOA transfers)                │    │
│  │  ├── URSTOA-Vault Submodule (Chain 0 only)                      │    │
│  │  │   ├── C_URV|Stake          (stake URSTOA)                    │    │
│  │  │   ├── C_URV|Unstake        (unstake URSTOA)                  │    │
│  │  │   └── C_URV|Collect        (claim STOA rewards)              │    │
│  │  └── A_InitialiseStoaChain    (genesis initialization)          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Core Services                                │    │
│  │  src/Chainweb/                                                  │    │
│  │  ├── Pact/PactService.hs  (Pact service orchestration)          │    │
│  │  ├── Version/StoaChain.hs (network definitions)                 │    │
│  │  └── GasPrice.hs          (dynamic gas pricing)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Documentation Index

### AncientStoa (Chainweb Node)

| Document | Description |
|----------|-------------|
| [**Full Technical README**](docs/chainweb-node/README.md) | Complete technical documentation with build instructions |
| [**🚀 Node Launch Checklist**](docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md) | **Pre-launch configuration guide** - Genesis time, keysets, bootstrap nodes |
| [**Yang Emission System**](docs/chainweb-node/EMISSION_SYSTEM.md) | Deterministic emission formula, 90/10 split, URSTOA-Vault, global supply registry |
| [**Yin Earnings (Gas)**](docs/chainweb-node/GAS_PRICE_SYSTEM.md) | Dynamic minimum gas price, time-based increases |
| [**Genesis System**](docs/chainweb-node/GENESIS_SYSTEM.md) | Genesis payload generation, transaction order, keysets |
| [**Pact 4 Removal**](docs/chainweb-node/PACT4_REMOVAL.md) | Detailed log of Pact 4 code removal |

### AncientPact (Pact 5.4.1 Fork)

| Document | Description |
|----------|-------------|
| [**Main README**](docs/pact-5/README.md) | AncientPact overview and StoaChain extensions |
| [**chain-data Extensions**](docs/pact-5/chain-data.md) | Documentation for `global-supply-register` and `external-fpa` fields |

---

## STOA Token Economics

### Emission Formula

STOA uses a **deterministic emission formula** instead of CSV-based allocations:

```
Daily Emission = (Ceiling - GlobalSupply) / EMISSION_SPEED
Block Emission = Daily Emission / BPD
```

Where:
- `Ceiling` = Initial ceiling (40× Genesis Supply), increases by 1M STOA annually
- `GlobalSupply` = Total STOA across all chains
- `EMISSION_SPEED` = 25,000 (divisor)
- `BPD` = 2,880 (blocks per day at 30-second intervals)

### Block Emission Split

| Recipient | Share | Description |
|-----------|-------|-------------|
| **Miner** | 90% | Direct block reward (Yang Emission) |
| **URSTOA-Vault** | 10% | Distributed to URSTOA stakers |

### Miner Income Sources

| Source | Name | Description |
|--------|------|-------------|
| Block Rewards | **Yang Emission** | Newly minted STOA (90% to miner, 10% to vault) |
| Transaction Fees | **Yin Earnings** | Gas fees transferred to miner (100%) |

### Genesis Supply

At genesis, **Chain 0** receives all initial supply via `A_InitialiseStoaChain`:
- **12M STOA** minted to the foundation account
- **1M URSTOA** minted to the foundation account
- **URSTOA-Vault** initialized with foundation as first staker

On Chains 1-9, the genesis transaction is a no-op.

### URSTOA Token

**URSTOA** (UR-STOA) is a secondary token that acts as a **perpetual virtual miner**, enabling holders to earn 10% of all Yang (block) emissions without performing any actual mining work.

| Property | Value |
|----------|-------|
| **Total Supply** | 1,000,000 URSTOA (fixed, minted at genesis) |
| **Precision** | 3 decimal places (0.001 URSTOA minimum) |
| **Chain Restriction** | **Chain 0 only** |
| **Purpose** | Staking to earn 10% of STOA emissions |

#### The Virtual Mining Concept

URSTOA represents **fractional ownership of perpetual mining rights**. By staking URSTOA in the Vault, holders become "virtual miners" who collectively receive 10% of every block's emissions—distributed proportionally based on their stake.

### URSTOA-Vault (Staking)

The **URSTOA-Vault** is a staking mechanism where URSTOA holders stake their tokens to earn their proportional share of the 10% foundation portion of block emissions.

#### How Virtual Mining Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Block Emission Flow                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Block Mined → Yang Emission Calculated                            │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │    90% → Miner Account    │                                │
│        └───────────────────────────┘                                │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │   10% → URSTOA-Vault      │  (Miner injects on Chain 0)    │
│        └───────────────────────────┘                                │
│                     │                                               │
│                     ▼                                               │
│        ┌───────────────────────────┐                                │
│        │ RPS Update → All Stakers  │  (Rewards accrue instantly)    │
│        │ earn proportional share   │                                │
│        └───────────────────────────┘                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### RPS (Reward Per Share) Mechanism

The Vault uses a **Reward Per Share** model for gas-efficient reward distribution:

1. When STOA is injected, `RPS += injected_amount / total_staked_urstoa`
2. User's pending rewards = `user_stake × (current_RPS - user_last_RPS)`
3. No loops needed—O(1) complexity regardless of staker count

### Chain-Data Extensions

StoaChain extends Pact's `chain-data` with two new fields (injected by the node before coinbase execution):

| Field | Type | Available On | Description |
|-------|------|--------------|-------------|
| `global-supply-register` | Decimal | All chains | Sum of `LocalSupply` from all chains |
| `external-fpa` | Decimal | Chain 0 only | Sum of `FoundationPending` from chains 1-9 |

---

## Governance

### Namespace Structure

StoaChain uses the `stoa-ns` namespace (replacing Kadena's `kadena` namespace):

```
stoa-ns/
├── foundation-keyset
├── stoa_master_one
├── stoa_master_two
├── stoa_master_three
├── stoa_master_four
├── stoa_master_five
├── stoa_master_six
└── stoa_master_seven
```

### 7 Stoa Masters

The STOA module is governed by 7 keysets. **Any one** of the 7 masters can authorize governance actions (module upgrades, etc.).

---

## Pre-Launch Configuration

### 🚀 Centralized Configuration (Recommended)

Instead of manually editing multiple files, use the **centralized configuration system**:

**Step 1: Edit the master config file**
```bash
nano stoachain-config.yaml
```

This single YAML file contains ALL settings:
- Genesis time
- Token economics (supply, ceiling)
- Foundation keyset (account + keys)
- 7 Stoa Masters keysets
- Namespace admin/operate keysets
- Gas price settings

**Step 2: Apply configuration to all files**
```bash
# Preview what will change
./scripts/apply-config.sh --dry-run

# Apply changes to all files
./scripts/apply-config.sh
```

**Step 3: Generate genesis payloads and build**
```bash
cd cwtools && cabal run ea
cd .. && cabal build chainweb-node
```

> 📋 **Full Documentation**: See [`docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md`](docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md) for the complete pre-launch guide.

---

## Source Repositories

> ⚠️ **Note**: The source repositories are currently private. This documentation repository provides public access to the project documentation.

- **AncientStoa** (Chainweb Node): StoaChain node implementation
- **AncientPact** (Pact 5.4.1): Pact smart contract language fork with StoaChain extensions

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

*Last Updated: December 2025*
