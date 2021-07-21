# Customer Database

## Overview

A customer in the zkChannels protocol may have a channel open with any number
of merchants. The customer database holds details about each channel.

### Channels Database

The channels database is used to remember details about a channel across
different sessions.

- **Label**: A text description input by the customer to identify a channel.
- **Address**: The `zkchannel://` address for the channel.
- **Initial Customer Balance**: The amount initially deposited by the customer
  upon establishing the channel. The customer may use this for their own
  accounting purposes, outside the scope of the protocol.
- **Initial Merchant Balance**: The amount initially deposited by the merchant
  upon establishing the channel. The customer may use this for their own
  accounting purposes, outside the scope of the protocol.
- **Channel State and Status**: All the information pertaining to the present state and status of the
  channel, including a unique channel identifier, allocation of the balance to customer and merchant, a nonce, and a revocation pair. The status must be one of the following: [Inactive][], [Ready][], [Started][],
  [Locked][], [PendingClose][], [Closed][]. Internal details of the channel state are discussed in the zkAbacus specification in [the zkChannels protocol document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf).

## Operations

| Name                                             | Input                                      | Output  | Summary                                                                                           |
| ------------------------------------------------ | ------------------------------------------ | ------- | ------------------------------------------------------------------------------------------------- |
| [**Insert Channel**][insert_channel]             | Label, Address, Inactive (All Required)    | None    | Insert the new channel if it doesn't already exist. If the insertion fails, return the Inactive. |
| [**Get Address**][get_address]                   | Label (Required)                           | Address | Get the address of a channel with a given label.                                                  |
| [**Update Channel Status**][update_channel_status] | Label, Old Status, New Status (All Required) | None    | Update an existing channel's status atomically.                                                    |

[insert_channel]: #insert-channel
[get_address]: #get-address
[relabel_channel]: #relabel-channel
[update_channel_status]: #update-channel-status

### Insert Channel

`zkAbacus.Establish` calls this operation. This operation is atomic to prevent
race conditions. For example, if multiple Establish sessions try to insert a
channel with the same label, only the first session succeeds. Subsequent inserts
fail because a channel already exists with the given label.

### Get Address

`zkAbacus.Pay` and `zkAbacus.Close` call this operation to connect to the
merchant.

### Update Channel Status

The customer uses this operation to progress the channel from one status to the
next. 
Each status transition is accompanied by a condition; the update function must be atomic and succeed if and only if the condition is met.

In a correct zkChannels execution, the statuses will transition in an expected order, based on the outcome of `zkAbacus` and `zkEscrowAgent` subprotocol executions. Establishing a channel includes the following status transitions:
```
Inactive -> Originated -> Customer funded -> Merchant funded -> Ready 
```
A payment on a channel goes through a status transition loop:
```
Ready -> Started -> Locked 
   ^                  |
   |__________________|
```

The closing procedure can begin from many statuses, but in a normal run, we expect it to start from the `Ready` status:
``` 
Ready -> Pending close -> Close
```

Here, we describe briefly what each status means for the channel:

| Status           | Condition | Notes |
| ---------------- | --------- | ----- |
| `Inactive`       | Initial channel state is agreed on. | The smart contract does not exist yet.
| `Originated`     | Contract is originated on chain.
| `CustomerFunded` | Contract is funded by the customer.
| `MerchantFunded` | Contract is funded by the merchant (possibly vacuously).
| `Ready`          | Customer has a valid pay token | Customer can make payments on the channel.
| `Started`        | Customer has made a payment request and provided proof that it is valid. | Customer cannot initiate another payment on the channel.
| `Locked`         | Customer has a valid closing signature for the new channel balances. | Customer cannot initiate another payment on the channel.
| `PendingClose`   | One of the parties has initiated a close procedure.
| `Closed`         | The balances on the smart contract are paid out. 


## Schema

**channels**

| field                    | properties       | type                                                                                   |
| ------------------------ | ---------------- | -------------------------------------------------------------------------------------- |
| label                    | unique, required | [ChannelName][channel_name]                                                            |
| address                  | required         | [ZkChannelAddress][zk_channel_address]                                                 |
| initial_customer_balance | required         | [CustomerBalance][customer_balance]                                                    |
| initial_merchant_balance | required         | [MerchantBalance][merchant_balance]                                                    |
| state                    | required         | [ChannelState][channel_state], which is one of: Inactive, Originated, CustomerFunded, MerchantFunded, Ready, Started, Locked, PendingClose, Closed |


[channel_state]: https://github.com/boltlabs-inc/zeekoe/blob/9240bbc0982c563be48d93df5c643dac3512614f/src/database/customer/state.rs#L15-L33
[channel_name]: https://github.com/boltlabs-inc/zeekoe/blob/9240bbc0982c563be48d93df5c643dac3512614f/src/customer.rs#L39-L41
[zk_channel_address]: https://github.com/boltlabs-inc/zeekoe/blob/9240bbc0982c563be48d93df5c643dac3512614f/src/transport/client.rs#L225-L231
[customer_balance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/states.rs#L194-L196
[merchant_balance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/states.rs#L158-L160
[inactive]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/customer.rs#L209-L217
[ready]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/customer.rs#L104-L113
[started]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/customer.rs#L339-L348
[locked]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/customer.rs#L479-L489
[pendingclose]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/customer.rs#L363-L370
[closed]: https://github.com/boltlabs-inc/zeekoe/blob/9240bbc0982c563be48d93df5c643dac3512614f/src/database/customer/state.rs#L37-L43

#### Notes

- **required:** This column cannot be empty.
- **unique:**  No two entries can contain the same value.