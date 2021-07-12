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
- **State**: All the information pertaining to the present state of the
  channel. One of the following: [Inactive][], [Ready][], [Started][],
  [Locked][], [PendingClose][], [Closed][].

## Operations

| Name                                             | Input                                      | Output  | Summary                                                                                           |
| ------------------------------------------------ | ------------------------------------------ | ------- | ------------------------------------------------------------------------------------------------- |
| [**Insert Channel**][insert_channel]             | Label, Address, Inactive (All Required)    | None    | Insert the new channel if it doesn't already exist. If the insertion failed, return the Inactive. |
| [**Get Address**][get_address]                   | Label (Required)                           | Address | Get the address of a channel with a given label.                                                  |
| [**Update Channel State**][update_channel_state] | Label, Old State, New State (All Required) | None    | Update an existing channel's state atomically.                                                    |

[insert_channel]: #insert-channel
[get_address]: #get-address
[relabel_channel]: #relabel-channel
[update_channel_state]: #update-channel-state

### Insert Channel

`zkAbacus.Establish` calls this operation. This operation is atomic to prevent
race conditions. For example, if multiple Establish sessions try to insert a
channel with the same lable, only the first will succeed. Subsequent inserts
will fail because a channel already exists with the label.

### Get Address

`zkAbacus.Pay` and `zkAbacus.Close` call this operation to connect to the
merchant.

### Update Channel State

The customer uses this operation to progress the channel from one state to the
next. Specifically:

| Step                 | Transition              | Condition                                                      |
| -------------------- | ----------------------- | -------------------------------------------------------------- |
| `zkAbacus.Establish` | `Inactive → Ready`      | Activated the pay token.                                       |
| `zkAbacus.Pay`       | `Ready → Started`       | Generated nonce and pay proof.                                 |
| `zkAbacus.Pay`       | `Started → Locked`      | Validated the closing signature.                               |
| `zkAbacus.Pay`       | `Locked → Ready`        | Validated the merchant's approval message.                     |
| `zkAbacus.Close`     | `PendingClose → Closed` | Mutual close has reached required confirmation depth on chain. |

This update should be performed within an atomic transaction, such that it only
succeeds if the condition is met.

## Schema

**channels**

| field                    | properties       | type                                                                                   |
| ------------------------ | ---------------- | -------------------------------------------------------------------------------------- |
| label                    | unique, required | [ChannelName][channel_name]                                                            |
| address                  | required         | [ZkChannelAddress][zk_channel_address]                                                 |
| initial_customer_balance | required         | [CustomerBalance][customer_balance]                                                    |
| initial_merchant_balance | required         | [MerchantBalance][merchant_balance]                                                    |
| state                    | required         | One of: [Inactive][], [Ready][], [Started][], [Locked][], [PendingClose][], [Closed][] |

[channel_name]: https://github.com/boltlabs-inc/zeekoe/blob/ace087b97f51752f8a86241772bb3eaf24c7ad3d/src/customer.rs#L39-L41
[zk_channel_address]: https://github.com/boltlabs-inc/zeekoe/blob/063d7275d65ed0c7cbe1c9d8b2f669fc86483978/src/transport/client.rs#L225-L231
[customer_balance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/0983f42dc9e8ecd08d6aa068ae9d24bee9913110/zkabacus-crypto/src/states.rs#L194-L196
[merchant_balance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/0983f42dc9e8ecd08d6aa068ae9d24bee9913110/zkabacus-crypto/src/states.rs#L158-L160
[inactive]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/99a77f534d0aace02b80fdd1ff141b5378c2c112/zkabacus-crypto/src/customer.rs#L209-L217
[ready]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/99a77f534d0aace02b80fdd1ff141b5378c2c112/zkabacus-crypto/src/customer.rs#L104-L113
[started]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/99a77f534d0aace02b80fdd1ff141b5378c2c112/zkabacus-crypto/src/customer.rs#L339-L348
[locked]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/99a77f534d0aace02b80fdd1ff141b5378c2c112/zkabacus-crypto/src/customer.rs#L479-L489
[pendingclose]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/99a77f534d0aace02b80fdd1ff141b5378c2c112/zkabacus-crypto/src/customer.rs#L363-L370
[closed]: https://github.com/boltlabs-inc/zeekoe/blob/b2272d2e47bda800082b399c01b6561cd7b165b5/src/database/customer/state.rs#L37-L43

#### Notes

- **required** - This column cannot be empty.
- **unique** - No two entries can contain the same value.
