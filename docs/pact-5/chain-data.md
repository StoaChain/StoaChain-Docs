## chain-data

Use `chain-data` to retrieve the blockchain-specific public metadata for a transaction. 
This function returns an object with the following fields:

- `chain-id`: The chain identifier (0-19) for the blockchain where the transaction was executed.
- `block-height`: The height of the block that includes the transaction.
- `block-time`: The timestamp of the block that includes the transaction.
- `prev-block-hash`: The hash of the previous block.
- `sender`: The sender of the transaction.
- `gas-limit`: The gas limit for the transaction.
- `gas-price`: The gas price for the transaction.
- `gas-fee`: The gas fee for the transaction.

### StoaChain Extensions (AncientPact 5.4.1)

> **Note**: The following fields are StoaChain-specific extensions and are only available when running on a StoaChain node. They are injected by the node runtime before coinbase execution.

- `global-supply-register`: (Decimal) The total STOA supply across all chains, computed by summing LocalSupply tables from all chains. Used for emission calculations.
- `external-fpa`: (Decimal, Chain 0 only) The sum of Foundation Pending Amounts accumulated on chains 1-9. Used by Chain 0's coinbase to calculate and mint the foundation share.

### Basic syntax

To retrieve the public metadata for a transaction using `chain-data`, use the following syntax:

```pact
(chain-data)
```

### Arguments

You can use the `chain-data` function without arguments in code that identifies the transaction that you want to return metadata for.

### Return value

The `chain-data` function returns the public metadata for a transaction as an object with the following fields

| Field | Type | Description
| ----- | ---- | -----------
| `chain-id` | string | The chain identifier (0-19) for the blockchain where the transaction was executed.
| `block-height` | integer | The height of the block that includes the transaction.
| `block-time` | time | The timestamp of the block that includes the transaction.
| `prev-block-hash` | string | The hash of the previous block.
| `sender` | string | The sender of the transaction.
| `gas-limit` | integer | The gas limit for the transaction.
| `gas-price` | decimal | The gas price for the transaction.
| `gas-fee` | decimal | The gas fee for the transaction.

#### StoaChain-Specific Fields

The following fields are available only on StoaChain nodes (AncientPact 5.4.1):

| Field | Type | Available On | Description
| ----- | ---- | ------------ | -----------
| `global-supply-register` | decimal | All chains | Sum of LocalSupply from all chains. Used in STOA emission calculation.
| `external-fpa` | decimal | Chain 0 only | Sum of FoundationPending amounts from chains 1-9. Used for foundation share minting.

### Examples

If you call the `chain-data` function in the Pact REPL without providing a transaction context in the surrounding code, the function returns the object with placeholder fields.
For example:

```pact
{"block-height": 0
,"block-time": "1970-01-01T00:00:00Z"
,"chain-id": ""
,"gas-limit": 0
,"gas-price": 0.0
,"prev-block-hash": ""
,"sender": ""}
```

If you provide context for the call, the function returns an object with fields similar to the following:

```pact
pact> (chain-data)
{
  "chain-id": "3",
  "block-height": 4357306,
  "block-time": "2024-06-06 20:12:56 UTC",
  "prev-block-hash": "33caae279bd584b655283b7d692d7e7b408d6549869c5eb6dcf2dc60021c3916",
  "sender": "k:1d5a5e10eb15355422ad66b6c12167bdbb23b1e1ef674ea032175d220b242ed4,
  "gas-limit": 2320,
  "gas-price": 1.9981e-7,
  "gas-fee": 726
}
```

In most cases, you use `chain-data` in Pact modules or in combination with frontend libraries to return information in the context of a specific transaction.
The following example illustrates using chain-data in a Pact module to get the block time from a transaction:

```pact
(let ((curr-time:time (at 'block-time (chain-data))))
```

### StoaChain Examples

On StoaChain, the `chain-data` object includes additional fields for emission calculations. These fields are injected by the node runtime during coinbase execution.

**Accessing global supply for emission calculation:**

```pact
;; Inside XM_StoaCoinbase - computes block reward based on remaining supply
(let
    (
        (current-total-supply:decimal (at "global-supply-register" (chain-data)))
        (current-ceiling:decimal (UC_CurrentCeiling))
        (remaining:decimal (- current-ceiling current-total-supply))
    )
    ;; Calculate emission based on remaining supply
    (floor (/ remaining (* BPD EMISSION-SPEED)) STOA_PREC)
)
```

**Accessing external FPA on Chain 0:**

```pact
;; Chain 0 coinbase reads accumulated FPA from chains 1-9
(let
    (
        (external-fpa:decimal (at "external-fpa" (chain-data)))
        (last-fpa:decimal (UR_LastFPA))
        (delta:decimal (- external-fpa last-fpa))
    )
    ;; delta represents new foundation share to mint
    delta
)
```

**REPL Testing with StoaChain fields:**

> ⚠️ **Important**: In REPL testing, `global-supply-register` and `external-fpa` do NOT exist unless explicitly mocked using `env-chain-data`.

```pact
;; Mock chain-data with StoaChain-specific fields for testing
(env-chain-data { 
    "chain-id": "0", 
    "block-time": (time "2026-06-15T12:00:00Z"),
    "global-supply-register": 100000000.0,
    "external-fpa": 350.0
})

;; Now functions that read these fields can be tested
(at "global-supply-register" (chain-data))  ;; Returns 100000000.0
(at "external-fpa" (chain-data))            ;; Returns 350.0
```

Without mocking, accessing these fields in REPL will fail with "Field not found in object".
