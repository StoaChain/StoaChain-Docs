# StoaChain Genesis Generation (Ea Tool)

> "Eä means 'to be' in Quenya, the ancient language of Tolkien's elves."
> — The Silmarillion

> **📖 Main Documentation**: See the [main README](../../README.md) for an overview of StoaChain.

This directory contains the **Ea** tool, which generates genesis block payloads for StoaChain networks.

## Overview

The genesis block (block height 0) is the foundational block of a blockchain. It contains the initial state of the chain, including:
- Namespace definitions
- Governance keysets
- The STOA token contract
- Gas payer contract

StoaChain supports **three networks**:

| Network | Version Name | Chains | Graph Type | Purpose |
|---------|--------------|--------|------------|---------|
| **Mainnet** | `stoamainnet01` | 10 | Petersen | Production network |
| **Testnet** | `stoatestnet02` | 3 | Triangle | Testing network |
| **Devnet** | `stoadevnet03` | 3 | Triangle | Development network |

## Directory Structure

```
chainweb-node/
├── cwtools/ea/
│   ├── Ea.hs              # Main genesis generator
│   ├── Ea/Genesis.hs      # Genesis transaction definitions
│   └── README.md          # This file
│
├── pact/
│   ├── genesis/
│   │   ├── mainnet/       # Mainnet genesis config (10 chains)
│   │   │   ├── ns.yaml              # Namespace configuration
│   │   │   └── stoa-masters.yaml    # 7 governance keysets
│   │   ├── testnet/       # Testnet genesis config (3 chains)
│   │   │   ├── ns.yaml
│   │   │   └── stoa-masters.yaml
│   │   └── devnet/        # Devnet genesis config (3 chains)
│   │       ├── ns.yaml
│   │       └── stoa-masters.yaml
│   │
│   ├── namespaces/
│   │   └── ns-install.pact    # Namespace module (creates stoa-ns, user, free)
│   │
│   ├── stoa-masters/
│   │   └── stoa-masters.pact  # 7 governance keyset definitions
│   │
│   ├── coin-contract/
│   │   ├── stoa.pact                  # STOA token module (includes URSTOA & Vault)
│   │   ├── stoa-install.pact          # Table creation (7 tables)
│   │   ├── stoa-initialise.yaml       # Genesis initialization
│   │   ├── load-stoa-interface.yaml   # Loads StoaFungibleV1
│   │   ├── load-stoa-contract.yaml    # Loads STOA module
│   │   ├── install-stoa-contract.yaml # Creates tables
│   │   └── v1/
│   │       ├── StoaFungibleV1.pact    # Token interface
│   │       └── stoa.pact              # Versioned STOA module
│   │
│   └── gas-payer/
│       └── load-gas-payer.yaml        # Gas payer contract
│
└── src/Chainweb/
    ├── Version/
    │   └── StoaChain.hs   # Network version definitions
    └── BlockHeader/Genesis/
        └── *.hs           # Generated payload modules (by Ea)
```

## Genesis Transaction Order

The genesis block executes transactions in the following order:

1. **Namespace Module** (`ns-install.pact`)
   - Defines the `ns` module for namespace management
   - Creates three namespaces:
     - `stoa-ns` - The main StoaChain namespace (controlled by ns-operate-keyset)
     - `user` - Open namespace for user contracts
     - `free` - Open namespace for free contracts

2. **Stoa Masters Keysets** (`stoa-masters.pact`)
   - Enters the `stoa-ns` namespace
   - Defines 7 governance keysets:
     - `stoa-ns.stoa_master_one` through `stoa-ns.stoa_master_seven`
   - These keysets control the STOA module governance

3. **StoaFungibleV1 Interface** (`StoaFungibleV1.pact`)
   - Defines the fungible token interface
   - Merged from Kadena's `fungible-v2` and `fungible-xchain-v1`

4. **STOA Module** (`stoa.pact`)
   - The main token contract
   - Implements `StoaFungibleV1`
   - Governed by the 7 Stoa Masters keysets (enforce-one)

5. **STOA Tables** (`stoa-install.pact`)
   - Creates `STOA.StoaTable` (STOA account balances)
   - Creates `STOA.LocalSupply` (per-chain STOA supply tracking)
   - Creates `STOA.FoundationPending` (10% Yang emission accumulation)
   - Creates `STOA.UR|StoaTable` (URSTOA account balances)
   - Creates `STOA.UR|LocalSupply` (URSTOA supply tracking)
   - Creates `STOA.URV|UrStoaVault` (URSTOA-Vault state)
   - Creates `STOA.URV|UrStoaVaultUser` (URSTOA-Vault user stakes)

6. **StoaChain Initialization** (`stoa-initialise.yaml`)
   - Calls `STOA.A_InitialiseStoaChain` with GENESIS capability
   - **On Chain 0:**
     - Creates foundation accounts for STOA and URSTOA
     - Mints `GENESIS-SUPPLY` (12M STOA) to foundation
     - Mints `URGENESIS-SUPPLY` (1M URSTOA) to foundation
     - Initializes URSTOA-Vault with foundation as first staker
   - **On Chains 1-9:**
     - No-op (returns success without doing anything)
   - All initial supply is minted exclusively on Chain 0

7. **Gas Payer** (`gas-payer-v1.pact`)
   - Enables gas station functionality

### GENESIS Capability

The node automatically grants the `GENESIS` capability during genesis block execution. This is handled in `src/Chainweb/Pact5/TransactionExec.hs`:

```haskell
[ CapToken (QualifiedName "GENESIS" (ModuleName "STOA" Nothing)) []
, CapToken (QualifiedName "COINBASE" (ModuleName "STOA" Nothing)) []
]
```

This allows the `XM_InitialMint` function to execute without external signatures.

## Keysets Explained

### Namespace Keysets (defined in ns.yaml)

| Keyset | Purpose |
|--------|---------|
| `ns-admin-keyset` | Governs the namespace module itself |
| `ns-operate-keyset` | Controls namespace operations (creating namespaces) |
| `ns-genesis-keyset` | Empty keyset used during genesis rotation |

### Stoa Masters Keysets (defined in stoa-masters.yaml)

The STOA module uses a **7-of-7 enforce-one** governance model:

```pact
(defcap GOV|STOA_MASTERS ()
    @event
    (enforce-one "Stoa Masters Permission not satisfied"
        [ (enforce-keyset "stoa-ns.stoa_master_one")
          (enforce-keyset "stoa-ns.stoa_master_two")
          ...
          (enforce-keyset "stoa-ns.stoa_master_seven")
        ]
    )
)
```

This means **any one** of the 7 master keysets can authorize governance actions.

## Pre-Launch Configuration

> ⚠️ **Before running the Ea tool**, ensure the following are configured:

### Required Constants in `pact/coin-contract/stoa.pact`:

| Constant | Description | Example |
|----------|-------------|---------|
| `GENESIS-SUPPLY` | Initial STOA supply | `12000000.0` |
| `GENESIS-CEILING` | Emission ceiling | `480000000.0` |
| `GENESIS-TIME` | Chain launch time | `"2026-01-01T00:00:00Z"` |

### Required Constants in `src/Chainweb/GasPrice.hs`:

| Constant | Description | Must Match |
|----------|-------------|------------|
| `stoaGenesisTime` | Genesis time (Haskell) | **MUST equal `GENESIS-TIME` in stoa.pact** |

> 🚨 **CRITICAL**: The `stoaGenesisTime` in Haskell and `GENESIS-TIME` in Pact **MUST be identical**.
> A mismatch will cause gas price validation failures. See [`docs/GAS_PRICE_SYSTEM.md`](../../docs/GAS_PRICE_SYSTEM.md).

### Required Keysets (in YAML files):

1. **Namespace keysets** in `pact/genesis/{network}/ns.yaml`:
   - `ns-admin-keyset`
   - `ns-operate-keyset`

2. **Stoa Masters keysets** in `pact/genesis/{network}/stoa-masters.yaml`:
   - `stoa_master_one` through `stoa_master_seven`

3. **Foundation keyset** in `pact/coin-contract/stoa-initialise.yaml`:
   - `stoa-foundation-keyset`

## Running the Ea Tool

```bash
# From the chainweb-node directory
cd cwtools
cabal run ea
```

This generates Haskell payload modules in `src/Chainweb/BlockHeader/Genesis/`:
- `StoaMainnet0to9Payload.hs` (10 chains)
- `StoaTestnet0to2Payload.hs` (3 chains)
- `StoaDevnet0to2Payload.hs` (3 chains)

## Chain Graph Configurations

### Petersen Graph (10 chains - Mainnet)

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

Each chain has exactly 3 neighbors (degree-3 regular graph).

### Triangle Graph (3 chains - Testnet/Devnet)

```
    0
   / \
  1---2
```

Each chain has exactly 2 neighbors.

## Differences from Kadena Chainweb

| Aspect | Kadena | StoaChain |
|--------|--------|-----------|
| Token Module | `coin` | `STOA` |
| Governance | `false` (always true) | 7 Stoa Masters keysets |
| Main Namespace | `kadena` | `stoa-ns` |
| Token Interface | `fungible-v2` + `fungible-xchain-v1` | `StoaFungibleV1` (merged) |
| Allocations | CSV-based vesting | None (emission-based) |
| Initial Keysets | 80+ investor keysets | 10 keysets (3 ns + 7 masters) |
| Pact Version | Pact 4 → Pact 5 migration | Pact 5 from genesis |
| Networks | mainnet01, testnet04, development, recap-development | stoamainnet01, stoatestnet02, stoadevnet03 |

## Version Folder Structure

The `src/Chainweb/Version/` folder contains network version definitions:

```
src/Chainweb/Version/
├── StoaChain.hs   # StoaChain network definitions (Mainnet, Testnet, Devnet)
├── Registry.hs    # Version lookup and registration
├── Guards.hs      # Fork activation guard functions
└── Utils.hs       # Utility functions for chain graphs and heights
```

**Removed Kadena files:**
- `Mainnet.hs` - Kadena Mainnet (not needed)
- `Testnet04.hs` - Kadena Testnet (not needed)
- `Development.hs` - Kadena Development (not needed)
- `RecapDevelopment.hs` - Kadena Recap Development (not needed)

## File Format (YAML)

Genesis YAML files have this structure:

```yaml
codeFile: path/to/pact-code.pact  # Pact code to execute
data:                              # Data available via (read-keyset ...)
  keyset-name:
    keys:
      - "public-key-hex"
    pred: keys-all                 # or keys-any, keys-2, etc.
nonce: unique-transaction-nonce
keyPairs: []                       # Empty for genesis (no signatures needed)
```

## Modifications Summary

### Files Created for StoaChain

1. **Version Definition**: `src/Chainweb/Version/StoaChain.hs`
   - Defines `stoaMainnet`, `stoaTestnet`, `stoaDevnet`
   - Pattern synonyms: `StoaMainnet01`, `StoaTestnet02`, `StoaDevnet03`

2. **Genesis Definitions**: `cwtools/ea/Ea/Genesis.hs`
   - `stoaMainnet0to9` (10 chains)
   - `stoaTestnet0to2` (3 chains)
   - `stoaDevnet0to2` (3 chains)

3. **Contract Loading YAMLs**:
   - `pact/coin-contract/load-stoa-interface.yaml`
   - `pact/coin-contract/load-stoa-contract.yaml`
   - `pact/coin-contract/install-stoa-contract.yaml`
   - `pact/coin-contract/stoa-initialise.yaml` (initializes StoaChain on Chain 0)

4. **Network-Specific Genesis Configs**:
   - `pact/genesis/mainnet/ns.yaml`
   - `pact/genesis/mainnet/stoa-masters.yaml`
   - `pact/genesis/testnet/ns.yaml`
   - `pact/genesis/testnet/stoa-masters.yaml`
   - `pact/genesis/devnet/ns.yaml`
   - `pact/genesis/devnet/stoa-masters.yaml`

### Files Removed (Kadena-specific)

- `pact/genesis/devnet/` (old Kadena devnet)
- `pact/genesis/testnet04/` (old Kadena testnet)
- `pact/genesis/testnet05/` (old Kadena testnet)
- `pact/genesis/mainnet/` (old Kadena mainnet with allocations)
- `pact/genesis/ns-v1.yaml`
- `pact/genesis/ns-v2.yaml`

### Registry Updates

`src/Chainweb/Version/Registry.hs` updated to include:
- `stoaMainnet`, `stoaTestnet`, `stoaDevnet` in `knownVersions`

## Next Steps After Genesis

After running the Ea tool:

1. Add generated payload modules to `chainweb.cabal`
2. Update `StoaChain.hs` to import and use actual payloads instead of `emptyPayload`
3. Configure bootstrap nodes for each network

---

## Global Supply Registry (Implemented)

The `XM_StoaCoinbase` function computes block rewards dynamically using two custom `chain-data` fields that are injected by the node runtime.

### Chain-Data Extensions

StoaChain uses **AncientPact** (a fork of Pact 5) with two additional `chain-data` fields:

| Field | Type | Available On | Description |
|-------|------|--------------|-------------|
| `global-supply-register` | Decimal | All chains | Sum of LocalSupply from all chains |
| `external-fpa` | Decimal | Chain 0 only | Sum of FoundationPending from chains 1-9 |

### Implementation

The implementation spans both repositories:

**AncientPact (Pact 5 Fork):**
- `Pact/Core/Evaluate.hs` - Added `_pdGlobalSupplyRegister` and `_pdExternalFPA` to `PublicData`
- `Pact/Core/IR/Eval/CEK/CoreBuiltin.hs` - Fields included in `coreChainData` output

**AncientStoa (Chainweb Node):**
- `src/Chainweb/Pact/GlobalSupply.hs` - Computes global supply and external FPA
- `src/Chainweb/Pact5/Types.hs` - `TxContext` includes `_tcGlobalSupply` and `_tcExternalFPA`
- `src/Chainweb/Pact5/TransactionExec.hs` - `ctxToPublicData` populates the fields

### How It Works

1. **Before coinbase execution**, the node:
   - Queries `LocalSupply` table on each chain and sums them → `global-supply-register`
   - Queries `FoundationPending` table on chains 1-9 and sums them → `external-fpa` (Chain 0 only)

2. **During coinbase**, the Pact code accesses these values:
   ```pact
   (current-total-supply:decimal (at "global-supply-register" (chain-data)))
   (external-fpa:decimal (at "external-fpa" (chain-data)))
   ```

3. **Timing**: Values are computed from the **previous** block state, ensuring deterministic computation across all nodes.

### Cross-Chain Supply Updates

When cross-chain transfers complete:
- Origin chain: `LocalSupply` decreases (via `X_UpdateLocalSupply amount false`)
- Target chain: `LocalSupply` increases (via `X_UpdateLocalSupply amount true`)

The global supply remains constant (tokens move, not created). The node's supply computation naturally handles this by summing all chains.

### REPL Testing

In the Pact REPL, these fields don't exist unless mocked:

```pact
(env-chain-data { 
    "chain-id": "0", 
    "block-time": (time "2026-06-15T12:00:00Z"),
    "global-supply-register": 100000000.0,
    "external-fpa": 350.0
})
```

See `pact/coin-contract/stoa.repl` for complete testing examples.

---

*Generated for StoaChain - A Kadena Chainweb Fork*
*Date: December 2025*

