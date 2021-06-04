## Customer Database

The customer database `customer_DB` is used to persist information across the steps of the protocol. This database needs to keep track of the following information:

1. The state of each channel - either the `Ready` struct or an enumeration of the three-step states `Ready`, `Started`, and `Locked`.
2. For each channel, the merchant `zkchannel://` address, which won't change for the lifetime of the channel.
3. A separate table of on-chain credentials - exact format TBD - based on the tezedge API's needs.

### `channels`

| cid | state | merchant_address | merchant_public_key | on chain credentials |
| --- | ----- | ---------------- | ------------------- | -------------------- |
| type? | Ready, Started, or Locked | everything after `zkchannel://` | TBD | TBD |

### Operations

TBD
