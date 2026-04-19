# StoaChain Genesis Generation (Ea Tool)

> "Eä means 'to be' in Quenya, the ancient language of Tolkien's elves."
> — The Silmarillion

> **📖 Main Documentation**: See the [main README](../../README.md) for an overview of StoaChain.

This directory contains the **Ea** tool, which generates genesis block payloads for StoaChain.

> ⚠️ **Corrections from earlier drafts (still applicable):**
>
> - **StoaChain is a single network**, not three. The live chain runs one `ChainwebVersion` called `stoa` (version code `0x0000000A`, 10 chains, Petersen graph, 30 s block delay) defined in `src/Chainweb/Version/Stoa.hs`.
> - **Genesis time** on the live chain is `2026-02-23T18:00:00.000000` UTC (`_genesisTime` in `src/Chainweb/Version/Stoa.hs`).
> - **Paths**: genesis Pact sources live under `pact/genesis/stoa/`; the STOA coin module is `pact/stoa-coin/new-coin.pact`.
> - **No `stoaGenesisTime` / `GENESIS-TIME` two-file sync**: the dynamic gas-price ramp has not shipped at the protocol level. See [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md).
> - **Pact is stock**: StoaChain uses unmodified upstream Pact 5.4. No `chain-data` extensions (no `global-supply-register`, no `external-fpa`), no `Chainweb.Pact.GlobalSupply` Haskell module. Emission is computed entirely inside the coin module using a calendar-based formula that doesn't need a global-supply aggregate per block.

## Overview

The genesis block (block height 0) is the foundational block of a blockchain. It contains the initial state of the chain, including:
- Namespace definitions
- Governance keysets
- The STOA token contract
- Gas payer contract

StoaChain ships a single network:

| Network | Version Name | Version Code | Chains | Graph | Block Delay |
|---------|--------------|--------------|--------|-------|-------------|
| **StoaChain** | `stoa` | `0x0000000A` (10) | 10 | Petersen | 30 s |

The `stoatestnet02` / `stoadevnet03` names that appeared in earlier drafts are not present in the node code. Local testing uses the same `stoa` version.

## Directory Structure

```
stoa-chain/
├── cwtools/ea/
│   ├── Ea.hs              # Main genesis generator
│   ├── Ea/Genesis.hs      # Genesis transaction definitions
│   └── README.md          # Ea reference
│
├── pact/
│   ├── genesis/stoa/      # Genesis payloads for the stoa network
│   │                      # (one YAML/pact set per chain, 0–9)
│   │
│   ├── namespaces/
│   │   └── ns-install.pact    # Namespace module (creates stoa-ns, user, free)
│   │
│   └── stoa-coin/
│       └── new-coin.pact      # STOA coin module (single file, used at genesis)
│
└── src/Chainweb/
    ├── Version/
    │   └── Stoa.hs       # stoa network version definition
    └── BlockHeader/Genesis/
        └── *.hs          # Generated payload modules (by Ea)
```

The `pact/coin-contract/v1/StoaFungibleV1.pact` / `pact/stoa-masters/` / `pact/gas-payer/` layouts shown in earlier drafts do not match the live tree — they described a planned rewrite that was not landed. Use `find` / `ls` against the node repo to get the current tree before documenting file-by-file structure.

## Genesis Transaction Order

The genesis block executes transactions in this order. Authoritative source is `pact/genesis/stoa/` plus `pact/stoa-coin/new-coin.pact` in the node repo; the sketch below describes the shape.

1. **Namespace module** (`pact/namespaces/ns-install.pact`)
   - Defines the `ns` module for namespace management.
   - Creates namespaces: `stoa-ns` (the main StoaChain namespace), `user`, `free`.

2. **Interfaces** (under `stoa-ns`)
   - `stoic-predicates` — custom keyset predicates (e.g., `5-of-9`).
   - `stoic-xchain` — autonomous accounts that pay for on-chain cross-chain transfers.
   - `stoic-fungible-v1` — StoaChain's equivalent of `fungible-v2`.
   - `ur-stoic-fungible-v1` — interface for the URSTOA token.
   - `fungible-v1` — identical to upstream `fungible-v2` (StoaChain just started at v1).
   - `fungible-xchain-v1` — identical to Kadena's.
   - `gas-payer-v1` — gas-station interface, identical to Kadena's.

3. **STOA coin module** (`pact/stoa-coin/new-coin.pact`)
   - Single module that defines both the STOA coin and the URSTOA token, plus the UrStoaVault state and RPS distribution logic.
   - Governed by **7 Stoa Masters keysets** (`stoa-ns.stoa_master_one` … `stoa-ns.stoa_master_seven`) with `enforce-one`.
   - Exposes `coin.URC_Emissions` (the per-block emission function returning `[block-emission urv-emission]`) and the supply table read/write paths.

4. **Genesis mint on Chain 0** (the genesis transaction is a no-op on chains 1-9)
   - **STOA — 16,000,000 total**: 10M ICO (held in the foundation account until the ICO finalises), 2M foundation, 4M Ouronet migration.
   - **URSTOA — 1,000,000 total**: 250k founders, 250k ICO sale (1 URSTOA per $5 contributed), 500k foundation.
   - The UrStoaVault is initialised on Chain 0.

### GENESIS Capability

The node automatically grants the `GENESIS` capability during genesis block execution, which allows the initial-mint functions in the coin module to execute without external signatures. The grant is performed in the Pact-5 transaction execution path (`src/Chainweb/Pact5/TransactionExec.hs`).

## Keysets Explained

### Namespace Keysets (defined in ns.yaml)

| Keyset | Purpose |
|--------|---------|
| `ns-admin-keyset` | Governs the namespace module itself |
| `ns-operate-keyset` | Controls namespace operations (creating namespaces) |
| `ns-genesis-keyset` | Empty keyset used during genesis rotation |

### Stoa Masters Keysets (coin-module governance)

The coin module is governed by **7 Stoa Masters keysets** — `stoa-ns.stoa_master_one` through `stoa-ns.stoa_master_seven` — combined with `enforce-one`, so **any 1-of-7** master keyset can authorise a governance action on the coin module:

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

Namespace administration (creating namespaces under `user` / `free`, etc.) uses a separate `ns-admin-keyset` / `ns-operate-keyset` pair defined in the genesis YAMLs — those gate namespace operations, not the coin module.

A `stoa-foundation` account holds the foundation's genesis STOA allocations (including the 10M held until the ICO finalises) and is controlled by a foundation keyset.

## Pre-Launch Configuration

> ⚠️ The "sync `stoaGenesisTime` (Haskell) with `GENESIS-TIME` (Pact)" requirement that appeared in earlier drafts **does not apply** to the live node. `src/Chainweb/GasPrice.hs` does not exist in the shipped code, and the coin module at `pact/stoa-coin/new-coin.pact` does not define `GENESIS-TIME`. The dynamic gas-price ramp is roadmap-only — see [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md).

The authoritative chain genesis time lives in `src/Chainweb/Version/Stoa.hs`:

```haskell
_genesisTime = 2026-02-23T18:00:00.000000 UTC
```

### Genesis supply

At genesis, Chain 0 receives all initial supply; chains 1-9 get no-ops.

**STOA — 16,000,000 total**:

| Allocation | Amount |
|------------|--------|
| ICO | 10,000,000 STOA |
| Foundation | 2,000,000 STOA |
| Ouronet migration | 4,000,000 STOA |

**URSTOA — 1,000,000 total** (Chain 0 only):

| Allocation | Amount |
|------------|--------|
| Founders | 250,000 URSTOA |
| ICO sale | 250,000 URSTOA |
| Foundation | 500,000 URSTOA |

### Keysets (in YAML files under `pact/genesis/stoa/`)

1. **Namespace keysets** (`ns.yaml`): `ns-admin-keyset`, `ns-operate-keyset` — gate namespace operations.
2. **Stoa Masters keysets**: `stoa_master_one` … `stoa_master_seven` — gate coin-module governance (1-of-7 enforce-one).
3. **Foundation keyset**: controls the `stoa-foundation` account that holds the foundation's STOA allocations, including the 10M held until ICO finalisation.

## Running the Ea Tool

```bash
# From the chainweb-node directory
cd cwtools
cabal run ea
```

This generates a Haskell payload module in `src/Chainweb/BlockHeader/Genesis/` for the 10 chains of the `stoa` network.

## Chain Graph

StoaChain uses the **Petersen graph** (10 chains, degree-3 regular):

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

Each chain has exactly 3 neighbors. There is no Triangle-graph testnet in the shipped node.

## Differences from Kadena Chainweb

| Aspect | Kadena | StoaChain |
|--------|--------|-----------|
| Token Module | `coin` | STOA coin module at `pact/stoa-coin/new-coin.pact` |
| Governance | `false` (always true) | 7 Stoa Masters keysets (`stoa_master_one` … `_seven`, `enforce-one`, any 1-of-7) govern the coin module |
| Main Namespace | `kadena` | `stoa-ns` |
| Fungible interface | `fungible-v2` + `fungible-xchain-v1` | `stoa-ns.stoic-fungible-v1` on the live chain (plus `ur-stoic-fungible-v1`, `fungible-v1`, `fungible-xchain-v1`) |
| Allocations | CSV-based vesting | None |
| Pact Version | Pact 4 → Pact 5 migration | Pact 5.4 on-chain; node still retains Pact 4 infrastructure internally (see [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md)) |
| Networks | mainnet01, testnet04, development, recap-development | single `stoa` network |

## Version Folder Structure

The `src/Chainweb/Version/` folder contains network version definitions:

```
src/Chainweb/Version/
├── Stoa.hs       # stoa network definition (single version)
├── Registry.hs   # Version lookup and registration
├── Guards.hs     # Fork activation guards
└── Utils.hs      # Utility functions
```

The upstream Kadena `Mainnet.hs` / `Testnet04.hs` / `Development.hs` / `RecapDevelopment.hs` files may still live in the repo — inspect the current tree to confirm what is retained vs. removed. Do not publish "removed files" lists without verifying them against the shipped code.

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

### Files actually shipped for StoaChain

1. **Version Definition**: `src/Chainweb/Version/Stoa.hs`
   - Defines the single `stoa` `ChainwebVersion` (version code `0x0000000A`, 10 chains, Petersen graph, 30 s block delay).
   - `_genesisTime = 2026-02-23T18:00:00.000000` UTC.
   - `_versionBootstraps` contains `node1.stoachain.com:1789`.

2. **Genesis Payload Sources**: `pact/genesis/stoa/` (per-chain YAML/pact files).

3. **STOA Coin Module**: `pact/stoa-coin/new-coin.pact` (single module used at genesis).

### Registry Updates

`src/Chainweb/Version/Registry.hs` registers `stoa` as a known version.

## Next Steps After Genesis

After running the Ea tool:

1. Add generated payload modules to `chainweb.cabal`.
2. Wire the payload into `src/Chainweb/Version/Stoa.hs`.
3. Confirm bootstrap configuration (`_versionBootstraps`) is correct for the intended environment.

---

## Pact is stock — no `chain-data` extensions

Earlier drafts of this doc described `global-supply-register` and `external-fpa` extensions to `chain-data`, plus a `Chainweb.Pact.GlobalSupply` Haskell module and a Foundation-Pending-Amount (FPA) delta-tracking mechanism. **None of that is in the live node.**

StoaChain uses **stock upstream Pact 5.4**. The Pact source-repository-package pin is declared in `cabal.project`. The emission formula in `coin.URC_Emissions` was rewritten into a linear calendar-based form precisely so that no global-supply aggregate is needed per block — which means no `chain-data` extension is needed either. Per-chain STOA supply is tracked in a Pact table inside the coin module; the explorer reads that table directly for live per-chain supply.

---

## Related Documentation

- **Emission System**: [`EMISSION_SYSTEM.md`](EMISSION_SYSTEM.md)
- **Gas Price System**: [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md)
- **Pact 4 retrospective**: [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md)

---

*Last updated: 2026-04-19*

