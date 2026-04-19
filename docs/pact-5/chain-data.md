## chain-data

> **Note — StoaChain runs stock Pact 5.4.** Earlier drafts of this document described two StoaChain-specific extensions (`global-supply-register` and `external-fpa`). **Those extensions are not present.** The StoaChain coin module's emission formula (`coin.URC_Emissions`) is calendar-based and does not need a global-supply aggregate at emission time, so no Pact-level extension was required. The documentation below describes the stock upstream `chain-data` native as it applies on StoaChain.

Use `chain-data` to retrieve the blockchain-specific public metadata for a transaction. This function returns an object with the following fields:

- `chain-id`: The chain identifier for the blockchain where the transaction was executed. On StoaChain this is `0`–`9` (10 chains, Petersen graph). On Kadena upstream it is `0`–`19`.
- `block-height`: The height of the block that includes the transaction.
- `block-time`: The timestamp of the block that includes the transaction.
- `prev-block-hash`: The hash of the previous block.
- `sender`: The sender of the transaction.
- `gas-limit`: The gas limit for the transaction.
- `gas-price`: The gas price for the transaction.
- `gas-fee`: The gas fee for the transaction.

### Basic syntax

```pact
(chain-data)
```

### Arguments

`chain-data` takes no arguments. It reads from the transaction context in which it is evaluated.

### Return value

The `chain-data` function returns the public metadata for a transaction as an object with the following fields:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `chain-id` | string | The chain identifier. On StoaChain: `0`–`9` (10 chains). On Kadena upstream: `0`–`19`. |
| `block-height` | integer | The height of the block that includes the transaction. |
| `block-time` | time | The timestamp of the block that includes the transaction. |
| `prev-block-hash` | string | The hash of the previous block. |
| `sender` | string | The sender of the transaction. |
| `gas-limit` | integer | The gas limit for the transaction. |
| `gas-price` | decimal | The gas price for the transaction. |
| `gas-fee` | decimal | The gas fee for the transaction. |

There are no StoaChain-specific fields. Any `chain-data` behavior documented by upstream Pact 5.4 applies verbatim.

### Examples

If you call `chain-data` in the Pact REPL without providing a transaction context in the surrounding code, the function returns an object with placeholder fields:

```pact
{"block-height": 0
,"block-time": "1970-01-01T00:00:00Z"
,"chain-id": ""
,"gas-limit": 0
,"gas-price": 0.0
,"prev-block-hash": ""
,"sender": ""}
```

With a real transaction context, the object looks like:

```pact
pact> (chain-data)
{
  "chain-id": "3",
  "block-height": 4357306,
  "block-time": "2024-06-06 20:12:56 UTC",
  "prev-block-hash": "33caae279bd584b655283b7d692d7e7b408d6549869c5eb6dcf2dc60021c3916",
  "sender": "k:1d5a5e10eb15355422ad66b6c12167bdbb23b1e1ef674ea032175d220b242ed4",
  "gas-limit": 2320,
  "gas-price": 1.9981e-7,
  "gas-fee": 726
}
```

Typical use in a Pact module — reading the block time of the containing transaction:

```pact
(let ((curr-time:time (at 'block-time (chain-data))))
    ...)
```

### Historical note

If you are porting StoaChain documentation from earlier drafts that reference `global-supply-register` or `external-fpa`, those references can be deleted. The coin module computes its emission without needing them, and no such fields exist in the Pact-5 build pinned by the node.
