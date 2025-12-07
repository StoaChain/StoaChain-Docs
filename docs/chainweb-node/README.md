# StoaChain

<p align="center">
<img src="docs/assets/StoaLogo.png" width="200" height="200" alt="StoaChain Logo" title="StoaChain">
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
- [Building from Source](#building-from-source)
- [Running a Node](#running-a-node)
- [Block Capacity & Throughput](#block-capacity--throughput)
- [STOA Token Economics](#stoa-token-economics)
  - [Emission Formula](#emission-formula)
  - [Genesis Supply](#genesis-supply)
  - [Supply Tracking & Global Supply Registry](#supply-tracking--global-supply-registry)
  - [URSTOA Token](#urstoa-token)
  - [URSTOA-Vault (Staking)](#urstoa-vault-staking)
  - [Chain-Data Extensions](#chain-data-extensions)
  - [REPL Testing Notes](#repl-testing-notes)
- [Governance](#governance)
- [Pre-Launch Configuration: Keys and Addresses](#pre-launch-configuration-keys-and-addresses)
- [Contributing](#contributing)
- [License](#license)

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
| **Networks** | mainnet01, testnet04, development, recap-development | stoamainnet01, stoatestnet02, stoadevnet03 |
| **Chain Count** | 20 chains (mainnet) | 10 chains (mainnet), 3 chains (testnet/devnet) |
| **Block Gas Limit** | 180k max, 150k default | 500k max, 400k default |
| **Gas Price** | Static minimum (1e-8 KDA) | Dynamic minimum (time-based) |

### StoaFungibleV1 Interface

StoaChain uses a simplified `StoaFungibleV1` interface that merges `fungible-v2` and `fungible-xchain-v1` into a single interface. A key simplification is the **removal of managed capabilities** from the TRANSFER function.

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

```pact
;; StoaChain TRANSFER capability - non-managed but fully validated
(defcap TRANSFER:bool (sender:string receiver:string amount:decimal)
    @event
    (UEV_Account sender)
    (UEV_Account receiver)
    (UEV_SenderWithReceiver sender receiver)
    (UEV_Amount amount "Transfer requires a positive amount")
    (UEV_StoaPrecision amount)
    (compose-capability (DEBIT sender amount))  ;; Enforces sender's guard
    (compose-capability (CREDIT receiver))
)
```

The validations inside the TRANSFER capability are sufficient to ensure transfers execute correctly and securely.

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
│  │  └── MinerReward.hs       (reward calculations)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Documentation Index

Detailed documentation is available in the following locations:

| Topic | Location | Description |
|-------|----------|-------------|
| 🚀 **Node Launch Checklist** | [`docs/NODE_LAUNCH_CHECKLIST.md`](docs/NODE_LAUNCH_CHECKLIST.md) | **Pre-launch configuration guide** - Genesis time, keysets, bootstrap |
| **Yang Emission System** | [`docs/EMISSION_SYSTEM.md`](docs/EMISSION_SYSTEM.md) | Deterministic emission, 90/10 split, URSTOA-Vault, global supply |
| **Yin Earnings (Gas)** | [`docs/GAS_PRICE_SYSTEM.md`](docs/GAS_PRICE_SYSTEM.md) | Dynamic minimum gas price, time-based increases |
| **Genesis System** | [`cwtools/ea/README.md`](cwtools/ea/README.md) | Genesis payload generation, transaction order, keysets |
| **Pact 4 Removal** | [`src/Chainweb/Pact4/README.md`](src/Chainweb/Pact4/README.md) | Detailed log of Pact 4 code removal |

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

**Haskell Toolchain:**
- GHC 9.6.x
- Cabal >= 3.8

Install via [GHCup](https://www.haskell.org/ghcup/):
```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
ghcup install ghc 9.6.4
ghcup install cabal 3.10.2.1
ghcup set ghc 9.6.4
```

### Build

```bash
# Clone the repository
git clone https://github.com/StoaChain/AncientStoa.git
cd AncientStoa/chainweb-node

# Update Cabal package index
cabal update

# Build the project
cabal build chainweb-node

# Find the binary location
cabal list-bin chainweb-node
```

### Generate Genesis Payloads

Before running a node, you need to generate the genesis payloads:

```bash
cd cwtools
cabal run ea
```

This generates Haskell modules in `src/Chainweb/BlockHeader/Genesis/`:
- `StoaMainnet0to9Payload.hs`
- `StoaTestnet0to2Payload.hs`
- `StoaDevnet0to2Payload.hs`

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
# Run a mainnet node
chainweb-node --chainweb-version=stoamainnet01

# Run a testnet node
chainweb-node --chainweb-version=stoatestnet02

# Run a devnet node (no PoW)
chainweb-node --chainweb-version=stoadevnet03
```

### Configuration

Generate a default configuration file:
```bash
chainweb-node --print-config > config.yaml
```

Key configuration options:
```yaml
chainweb:
  chainwebVersion: stoamainnet01
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
curl -sk "https://localhost:1789/chainweb/0.0/stoamainnet01/cut"
```

---

## Block Capacity & Throughput

StoaChain significantly increases block capacity compared to Kadena, leveraging Pact 5's improved performance.

### Gas Limits

| Parameter | Kadena | StoaChain | Improvement |
|-----------|--------|-----------|-------------|
| **Protocol Maximum** | 180,000 | 500,000 | +178% |
| **Default Config** | 150,000 | 400,000 | +167% |

### Throughput Comparison

| Metric | Kadena Mainnet | StoaChain Mainnet |
|--------|----------------|-------------------|
| Chains | 20 | 10 |
| Gas/Block (default) | 150,000 | 400,000 |
| Gas/Block (max) | 180,000 | 500,000 |
| **Total Capacity/30s (default)** | 3,000,000 | 4,000,000 |
| **Total Capacity/30s (max)** | 3,600,000 | 5,000,000 |
| **Improvement** | Baseline | **+33% to +39%** |

> Even with half the chains, StoaChain has **significantly higher total throughput** than Kadena.

### Why This is Safe

Pact 5 executes at approximately **2.5 microseconds per gas unit**:

| Gas Limit | Execution Time | Block Time | Headroom |
|-----------|----------------|------------|----------|
| 400,000 (default) | 1,000ms | 30,000ms | 30× |
| 500,000 (maximum) | 1,250ms | 30,000ms | 24× |

The 30-second block time provides ample headroom for even maximum-capacity blocks.

### Configuring Gas Limit

Node operators can adjust the gas limit via:

**Config file:**
```yaml
gasLimitOfBlock: 400000  # Up to 500000
```

**Command line:**
```bash
chainweb-node --block-gas-limit=450000
```

---

## STOA Token Economics

### Emission Formula

STOA uses a **deterministic emission formula** instead of CSV-based allocations:

```
Daily Emission = (Ceiling - GlobalSupply) / EMISSION_SPEED
Block Emission = Daily Emission / BPD
```

Where:
- `Ceiling` = Initial ceiling (40× Genesis Supply), increases by 1M STOA annually on January 1st at 00:00 UTC
- `GlobalSupply` = Total STOA across all chains
- `EMISSION_SPEED` = 25,000 (divisor)
- `BPD` = 2,880 (blocks per day at 30-second intervals)

### Dynamic Ceiling

The ceiling increases by **1 million STOA** at the start of each year. This is computed dynamically in the `UC_CurrentCeiling` function based on `block-time`:

```pact
(defun UC_CurrentCeiling ()
    (let
        (
            (genesis-year:decimal (UC_GetYear GENESIS-TIME))
            (year-now:decimal (UC_GetYear (at "block-time" (chain-data))))
            (new-millions:decimal (* 1000000.0 (- year-now genesis-year)))
        )
        (+ new-millions GENESIS-CEILING)
    )
)
```

### Genesis Supply

> ⚠️ **Placeholder Values**: The current code uses **12,000,000 STOA** as genesis supply and **480,000,000 STOA** (40× genesis) as initial ceiling. These are placeholders. The actual genesis supply will be determined before mainnet launch and will be somewhere between **12-15 million STOA**. The initial ceiling will be 40× the final genesis supply. These values will be updated in `stoa.pact` before chain start.

At genesis, **Chain 0** receives all initial supply via `A_InitialiseStoaChain`:
- **12M STOA** minted to the foundation account
- **1M URSTOA** minted to the foundation account (see [URSTOA Token](#urstoa-token) below)
- **URSTOA-Vault** initialized with foundation as first staker

On Chains 1-9, the genesis transaction is a no-op (STOA can be transferred there via `C_TransferAcross`).

### Supply Tracking & Global Supply Registry

Each chain maintains a `LocalSupply` table tracking STOA on that chain. The **global supply** is computed by summing all chains' local supplies before each coinbase transaction.

```pact
;; Query local supply on a chain
(STOA.UR_LocalStoaSupply)

;; Global supply is available in chain-data (during coinbase)
(at "global-supply-register" (chain-data))
```

The global supply registry is implemented via:
1. **AncientPact** (Pact-5 fork): Extended `chain-data` to include `global-supply-register` field
2. **AncientStoa** (Chainweb fork): `Chainweb.Pact.GlobalSupply` module queries all chains' `LocalSupply` tables

See [`docs/EMISSION_SYSTEM.md`](docs/EMISSION_SYSTEM.md) for detailed implementation documentation.

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

**Why 1 Million URSTOA?**
- The 1M supply with 3-decimal precision allows for **extremely granular** distribution of the 10% foundation share
- Each 0.001 URSTOA represents 1 billionth of the total virtual mining power
- This enables precise reward distribution even for very small stake positions

**Key Functions:**
```pact
;; Query URSTOA balance
(STOA.UR_UR|Balance "account")

;; Transfer URSTOA (Chain 0 only)
(STOA.C_UR|Transfer "sender" "receiver" 100.0)

;; Transfer with create-if-not-exists
(STOA.C_UR|TransferAnew "sender" "receiver" receiver-guard 100.0)
```

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

**Granularity Example:**
- Total URSTOA staked: 500,000 URSTOA
- User stake: 0.001 URSTOA (minimum)
- User's share: 0.000002% of the 10% foundation share
- If block emission is 5.0 STOA:
  - Foundation share: 0.5 STOA
  - User's share: 0.00000001 STOA (with 12-decimal STOA precision)

#### Vault Functions

```pact
;; Stake URSTOA in the vault
(STOA.C_URV|Stake "account" 1000.0)

;; Unstake URSTOA from the vault (must leave at least 1.0 URSTOA total in vault)
(STOA.C_URV|Unstake "account" 500.0)

;; Collect accumulated STOA rewards
(STOA.C_URV|Collect "account")

;; Query vault info
(STOA.UR_URV|VaultUrSupply)      ;; Total URSTOA staked
(STOA.UR_URV|VaultSupply)        ;; Total STOA rewards available
(STOA.UR_URV|UserSupply "account") ;; User's staked URSTOA
(STOA.URC_AvailableRewards "account") ;; User's claimable rewards
```

### Foundation Pending Amount (FPA)

On Chains 1-9, the 10% foundation share is **accumulated** in a `FoundationPending` table (not minted). Chain 0 reads this accumulated amount via `external-fpa` and mints the delta.

```pact
;; Query FPA on chains 1-9
(STOA.UR_FPA)

;; External FPA is available in chain-data on Chain 0 (during coinbase)
(at "external-fpa" (chain-data))
```

### Chain-Data Extensions

StoaChain extends Pact's `chain-data` with two new fields (injected by the node before coinbase execution):

| Field | Type | Available On | Description |
|-------|------|--------------|-------------|
| `global-supply-register` | Decimal | All chains | Sum of `LocalSupply` from all chains |
| `external-fpa` | Decimal | Chain 0 only | Sum of `FoundationPending` from chains 1-9 |

**Implementation:**
- **Haskell side**: `Chainweb.Pact.GlobalSupply` module computes values
- **Pact side**: Values injected into `PublicData` and available via `(chain-data)`

### REPL Testing Notes

> ⚠️ **Important**: When testing in the Pact REPL, `global-supply-register` and `external-fpa` fields do **NOT** exist in `chain-data` unless you explicitly mock them.

The node runtime injects these values during actual block execution. In REPL testing:

```pact
;; Mock chain-data with custom fields for testing
(env-chain-data { 
    "chain-id": "0", 
    "block-time": (time "2026-06-15T12:00:00Z"),
    "global-supply-register": 100000000.0,
    "external-fpa": 350.0
})

;; Now XM_StoaCoinbase can be tested
(test-capability (COINBASE))
(XM_StoaCoinbase "miner" miner-guard)
```

Without mocking, accessing these fields will fail:
```pact
;; This will fail in REPL without mocking:
(at "global-supply-register" (chain-data))
;; Error: Field not found in object
```

See `pact/coin-contract/stoa.repl` for complete testing examples.

---

## Governance

### Namespace Structure

StoaChain uses the `stoa-ns` namespace (replacing Kadena's `kadena` namespace):

```
stoa-ns/
├── stoa_master_one
├── stoa_master_two
├── stoa_master_three
├── stoa_master_four
├── stoa_master_five
├── stoa_master_six
└── stoa_master_seven
```

### 7 Stoa Masters

The STOA module is governed by 7 keysets using `enforce-one`:

```pact
(defcap GOV|STOA_MASTERS ()
    @event
    (enforce-one "Stoa Masters Permission not satisfied"
        [ (enforce-keyset "stoa-ns.stoa_master_one")
          (enforce-keyset "stoa-ns.stoa_master_two")
          (enforce-keyset "stoa-ns.stoa_master_three")
          (enforce-keyset "stoa-ns.stoa_master_four")
          (enforce-keyset "stoa-ns.stoa_master_five")
          (enforce-keyset "stoa-ns.stoa_master_six")
          (enforce-keyset "stoa-ns.stoa_master_seven")
        ]
    )
)
```

**Any one** of the 7 masters can authorize governance actions (module upgrades, etc.).

---

## Pre-Launch Configuration: Keys and Addresses

> ⚠️ **IMPORTANT**: Before launching StoaChain, you must configure the actual public keys in the genesis YAML files. The current files contain **placeholder keys** that must be replaced with real keys.

### Keys to Configure

There are **10 keysets/addresses** that need to be configured across the genesis YAML files:

| # | Keyset | Purpose | File Location |
|---|--------|---------|---------------|
| 1 | `ns-admin-keyset` | Governs the namespace module | `pact/genesis/{network}/ns.yaml` |
| 2 | `ns-operate-keyset` | Controls namespace operations | `pact/genesis/{network}/ns.yaml` |
| 3 | `stoa_master_one` | STOA module governance (1/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 4 | `stoa_master_two` | STOA module governance (2/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 5 | `stoa_master_three` | STOA module governance (3/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 6 | `stoa_master_four` | STOA module governance (4/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 7 | `stoa_master_five` | STOA module governance (5/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 8 | `stoa_master_six` | STOA module governance (6/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 9 | `stoa_master_seven` | STOA module governance (7/7) | `pact/genesis/{network}/stoa-masters.yaml` |
| 10 | `stoa-foundation-keyset` | Receives genesis supply (STOA + URSTOA) | `pact/coin-contract/stoa-initialise.yaml` |

> **Note**: `ns-genesis-keyset` is intentionally empty (used for genesis rotation only).

### File Locations by Network

```
pact/
├── coin-contract/
│   └── stoa-initialise.yaml       # Foundation address (receives STOA + URSTOA)
│
└── genesis/
    ├── mainnet/
    │   ├── ns.yaml                 # ns-admin-keyset, ns-operate-keyset
    │   └── stoa-masters.yaml       # 7 Stoa Masters keysets
    │
    ├── testnet/
    │   ├── ns.yaml
    │   └── stoa-masters.yaml
    │
    └── devnet/
        ├── ns.yaml
        └── stoa-masters.yaml
```

### YAML Key Format

Each keyset in the YAML files follows this format:

```yaml
keyset-name:
  keys:
    - "public-key-hex-string-1"
    - "public-key-hex-string-2"  # Optional: for multi-sig
  pred: keys-all  # or keys-any, keys-2, etc.
```

### Example: Configuring the Foundation Address

Edit `pact/coin-contract/stoa-initialise.yaml`:

```yaml
code: |-
  (STOA.A_InitialiseStoaChain "stoa-foundation" (read-keyset "stoa-foundation-keyset"))

data:
  stoa-foundation-keyset:
    keys:
      - YOUR_FOUNDATION_PUBLIC_KEY_HERE  # Replace with actual key
    pred: keys-all

nonce: stoa-initialise-genesis
keyPairs: []
```

> **Note**: `A_InitialiseStoaChain` handles all genesis initialization on Chain 0:
> - Creates foundation STOA and URSTOA accounts
> - Mints 12M STOA and 1M URSTOA
> - Initializes the URSTOA-Vault with foundation as first staker
> 
> On Chains 1-9, this transaction is a successful no-op.

### Pre-Launch Checklist

Before starting the node for the first time:

#### STOA Token Configuration
- [ ] Update `GENESIS-SUPPLY` in `pact/coin-contract/stoa.pact` (12-15M)
- [ ] Update `GENESIS-CEILING` in `pact/coin-contract/stoa.pact` (40× supply)

#### Genesis Time Configuration (MUST MATCH!)
- [ ] Update `GENESIS-TIME` in `pact/coin-contract/stoa.pact` (launch date)
- [ ] Update `stoaGenesisTime` in `src/Chainweb/GasPrice.hs` **(MUST match above!)**

> ⚠️ **CRITICAL**: The `GENESIS-TIME` constant in Pact and `stoaGenesisTime` in Haskell 
> **MUST be identical**. Mismatched values will cause gas price validation failures.
> See [`docs/GAS_PRICE_SYSTEM.md`](docs/GAS_PRICE_SYSTEM.md) for details.

#### Keyset Configuration
- [ ] Configure `ns-admin-keyset` and `ns-operate-keyset` in `ns.yaml`
- [ ] Configure all 7 `stoa_master_*` keysets in `stoa-masters.yaml`
- [ ] Configure `stoa-foundation-keyset` in `stoa-initialise.yaml`

#### Build & Deploy
- [ ] Run `cabal run ea` to regenerate genesis payloads
- [ ] Rebuild the node with new payloads

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
- **GitHub Repository**: [https://github.com/StoaChain/AncientStoa](https://github.com/StoaChain/AncientStoa)

---

*StoaChain - Building the future of decentralized computing*

*Last Updated: December 2025*
