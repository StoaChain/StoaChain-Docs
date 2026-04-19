# StoaChain Yin Earnings System (Dynamic Gas Price)

This document describes the **Yin Earnings** mechanism — the gas-based income that miners earn from transaction execution.

> ⚠️ **Not yet wired to the protocol.** The time-based dynamic minimum gas price (10,000 ANU → +1 ANU every 3 hours, capped at 400,000 ANU) is the **target design**. It is not yet enforced at the mempool / block-validation level on the live chain.
>
> **What the live chain does today:** it uses the upstream chainweb-node minimum gas price (`_configMinGasPrice = 1e-8`, defined in `src/Chainweb/Chainweb/Configuration.hs`). There is no per-block ramp applied by the Haskell mempool or validator.
>
> **What the coin module does today:** the coin module contains a Pact function that computes the intended ramp value (the design described below). It's usable from Pact, but nothing in the node's mempool-insertion or block-validation code path consults it as a floor yet.
>
> Wiring this ramp into the protocol is a planned upgrade. No target date has been published. Read the file paths and Haskell snippets below as design sketches for that wiring — the Haskell module they describe does not exist today.

---

## Yin vs Yang: Miner Income Sources

StoaChain miners earn STOA from two distinct sources:

| Source | Name | Description | Document |
|--------|------|-------------|----------|
| **Block Rewards** | **Yang Emission** | Newly minted STOA (90% to miner) | `EMISSION_SYSTEM.md` |
| **Transaction Fees** | **Yin Earnings** | Gas fees from transactions (100% to miner) | **This document** |

### Key Difference

```
YANG EMISSION                        YIN EARNINGS
─────────────────────────────────    ─────────────────────────────────
• Newly minted STOA                  • Transfer of existing STOA
• Decreases over time                • Increases over time (min price)
• 90% miner / 10% foundation         • 100% to miner
• Supply-based formula               • Activity-based (tx volume)
• Inflationary                       • Non-inflationary
```

Unlike Yang Emission (which mints new tokens), **Yin Earnings are not emissions** - they represent the transfer of existing STOA from transaction senders to miners as payment for block space.

---

## Overview (target design — Pact-side exists, protocol wiring pending)

The target **time-based dynamic minimum gas price**:
- Starts at **10,000 ANU** at genesis
- Increases by **1 ANU** every **3 hours**
- Caps at **400,000 ANU** (reached after ~133 years)

The coin module already contains a Pact function that computes this value as a function of block time. The live chain does **not** use it as an enforced minimum — mempool insertion and block validation still apply the upstream static `1e-8` floor. Wiring the Pact-computed value into the protocol-level minimum check is a planned upgrade.

### ANU (Atomic Unit)

**ANU** is the smallest unit of STOA:
```
1 STOA = 10^12 ANU = 1,000,000,000,000 ANU
```

STOA has 12 decimal places of precision, matching Kadena's KDA.

### Gas Price Conversion

| ANU | STOA |
|-----|------|
| 10,000 | 0.00000001 (1e-8) |
| 100,000 | 0.0000001 (1e-7) |
| 400,000 | 0.0000004 (4e-7) |

---

## Formula

```
If time <= genesis_time:
    minimum_gas_price = 10,000 ANU

Else:
    seconds_elapsed = current_time - genesis_time
    intervals = floor(seconds_elapsed / 10,800)  # 10,800 = 3 hours
    minimum_gas_price = min(10,000 + intervals, 400,000)
```

### Timeline

| Time After Genesis | Minimum Gas Price (ANU) | Minimum Gas Price (STOA) |
|--------------------|-------------------------|--------------------------|
| 0 hours | 10,000 | 0.00000001 |
| 3 hours | 10,001 | 0.000000010001 |
| 24 hours | 10,008 | 0.000000010008 |
| 1 week | 10,056 | 0.000000010056 |
| 1 month | 10,240 | 0.00000001024 |
| 1 year | 12,920 | 0.00000001292 |
| 10 years | 39,200 | 0.0000000392 |
| 133.5 years | 400,000 (cap) | 0.0000004 |

---

## Configuration

### Pact side — exists, callable, not enforced

The coin module (`pact/stoa-coin/new-coin.pact`) contains a function that computes the target minimum gas price as a function of block time (starting at 10,000 ANU, stepping by 1 ANU every 3 hours, capped at 400,000 ANU). You can call it from Pact to see the value the ramp *would* produce. The authoritative chain genesis time used for the ramp is the chain's `_genesisTime` (`2026-02-23T18:00:00.000000` UTC, defined in `src/Chainweb/Version/Stoa.hs`).

What's missing is the enforcement wiring: the mempool and block-validation paths still use the upstream static `_configMinGasPrice = 1e-8` as the floor. The Pact-computed ramp value is not consulted.

### Haskell side — sketch for the wiring upgrade

Earlier drafts described a `src/Chainweb/GasPrice.hs` module with a `stoaGenesisTime` constant and an `assertMinGasPrice` validator. **That module does not exist today.** The shape a future wiring upgrade might take:

```haskell
-- Hypothetical src/Chainweb/GasPrice.hs (not in the shipped code)
stoaGenesisTime :: UTCTime
stoaGenesisTime = parseTimeOrError True defaultTimeLocale "%Y-%m-%dT%H:%M:%SZ"
    "2026-02-23T18:00:00Z"

genesisMinGasPriceANU :: Integer
genesisMinGasPriceANU = 10_000

maxGasPriceANU :: Integer
maxGasPriceANU = 400_000

gasPriceIntervalSeconds :: Integer
gasPriceIntervalSeconds = 10_800  -- 3 hours
```

---

## Implementation Details

### Enforcement Points

The minimum gas price is enforced at **two points**:

#### 1. Mempool Insertion (`InMem.hs`)

When a transaction is submitted to the mempool:

```haskell
gasPriceMinCheck :: Either InsertError ()
gasPriceMinCheck = ebool_ (InsertErrorUndersized txPrice minPrice) (txPrice >= minPrice)
  where
    -- Use TX creation time for minimum calculation
    Time (TimeSpan (Micros creationTimeMicros)) = txMetaCreationTime $ txMetadata txcfg t
    minPrice = computeMinGasPriceFromCreationTime creationTimeMicros
```

#### 2. Block Validation (`Validations.hs`)

When validating transactions in a block:

```haskell
assertMinGasPrice :: P.GasPrice -> P.TxCreationTime -> Bool
assertMinGasPrice txGasPrice (P.TxCreationTime creationTimeSecs) =
    txGasPrice >= computeMinGasPriceFromCreationTime (creationTimeSecs * 1_000_000)
```

### Why TX Creation Time?

Both enforcement points use the **transaction's creation time** (not current time) to compute the minimum. This ensures:

1. **Consistency**: Mempool and block validation always agree
2. **Lenient Policy**: A TX valid at submission stays valid until expiry
3. **No Manipulation**: TTL check prevents fake creation times

### Validation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Transaction Lifecycle                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User creates TX                                             │
│     ├── creationTime = T                                        │
│     └── gasPrice = X                                            │
│                                                                 │
│  2. TX submitted to mempool                                     │
│     ├── TTL check: Is T recent? (within ~2 minutes)            │
│     ├── Gas price check: Is X >= computeMinGasPrice(T)?        │
│     │   ├── YES → Accept into mempool                          │
│     │   └── NO  → Reject: InsertErrorUndersized                │
│     │                                                          │
│  3. TX included in block (possibly hours later)                │
│     ├── Block validation uses same check                       │
│     ├── computeMinGasPrice(T) → same result                    │
│     └── TX still valid!                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pact Query Functions

Users can query the current minimum gas price via Pact:

### Get Minimum in ANU

```pact
(STOA.UC_MinimumGasPriceANU)
;; Returns: 10042 (example, depends on current time)
```

### Get Minimum in STOA

```pact
(STOA.UC_MinimumGasPriceSTOA)
;; Returns: 0.000000010042 (example)
```

---

## Upstream behavior retained on the live chain

Earlier drafts of this document claimed that the static-gas-price machinery had been removed from the node. **That is not the case.** The live node keeps the upstream chainweb-node static gas price pathway intact:

- `src/Chainweb/Chainweb/Configuration.hs` still defines `_configMinGasPrice` (default `1e-8`).
- Mempool insertion and block validation use that static floor.
- There is no `src/Chainweb/GasPrice.hs` module with ramp logic, no `stoaGenesisTime`, no `assertMinGasPrice` that consults creation time.

Any "Removed: `_configMinGasPrice` field" claim in earlier drafts was incorrect — do not reproduce it.

---

## Files that would change if the ramp ships (design sketch)

These paths describe a hypothetical future implementation, not committed code:

| File | Hypothetical change |
|------|---------------------|
| **NEW** `src/Chainweb/GasPrice.hs` | Dynamic gas price computation module |
| `src/Chainweb/Mempool/InMem.hs` | Use dynamic minimum based on TX creation time |
| `src/Chainweb/Pact5/Validations.hs` | Add an `assertMinGasPrice` validation |
| `pact/stoa-coin/new-coin.pact` | Add gas-price constants and query functions |
| `chainweb.cabal` | Expose the new `Chainweb.GasPrice` module |

---

## Troubleshooting

### Error: "Gas price below minimum for transaction creation time"

**Cause**: The transaction's gas price is below the minimum for its creation time.

**Solution**: 
1. Query the current minimum: `(STOA.UC_MinimumGasPriceSTOA)`
2. Set your transaction's gas price to at least this value
3. Add a small buffer to account for time passing

### Error: "InsertErrorUndersized"

**Cause**: Mempool rejected the transaction due to low gas price.

**Solution**: Same as above - use the current minimum gas price or higher.

### Genesis Time Mismatch (obsolete — ramp not shipped)

The two-file-sync requirement does not apply to the live chain. There is only one authoritative genesis time (in `src/Chainweb/Version/Stoa.hs`), and the static `1e-8` floor does not depend on it.

---

## Comparison with Kadena

| Aspect | Kadena | StoaChain (today) | StoaChain (proposed) |
|--------|--------|-------------------|----------------------|
| Minimum Gas Price | Static (`1e-8`) | Static (`1e-8`, upstream default) | Dynamic (time-based) |
| Configuration | CLI/config file | CLI/config file (upstream) | Hardcoded + Pact |
| Starting Price | Fixed | Fixed at `1e-8` | 10,000 ANU |
| Price Increase | None | None | +1 ANU every 3 hours |
| Maximum Price | None | None | 400,000 ANU |
| Query Function | N/A | N/A | `UC_MinimumGasPriceSTOA` |

---

## Related Documentation

- **Yang Emission**: [`EMISSION_SYSTEM.md`](EMISSION_SYSTEM.md) — Deterministic block rewards (90/10 split)
- **Genesis System**: [`GENESIS_SYSTEM.md`](GENESIS_SYSTEM.md) — Genesis payload layout
- **Pact 4 retrospective**: [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md)

---

*Last updated: 2026-04-19*

