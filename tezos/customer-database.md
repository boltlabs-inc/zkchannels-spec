# Customer Database

## Overview

A customer in the zkChannels protocol may have a channel open with any number
of merchants. The customer database holds details about each channel.

### Channels Database

The channels database is used to remember details about a channel across
different sessions. We describe the minimal information a customer should store about their channels.

- **Label**: A text description input by the customer to identify a channel.
- **Address**: The `zkchannel://` address for the channel.
- **Merchant Deposit**, **Customer Deposit**: The amount initially deposited by the merchant (`init_merchant_balance`) and the customer (`init_customer_balance`), respectively, upon establishing the channel. The customer may use this for their own accounting purposes, outside the scope of the protocol.
- **Channel State and Status**: All the information pertaining to the present state and status of the
  channel, including a unique channel identifier, allocation of the balance to customer and merchant, a nonce, and a revocation pair. The set of valid statuses are listed in the table below.
  Internal details of the channel state are discussed in the zkAbacus specification in [the zkChannels protocol document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf).
- **Closing Balances**: This field is set when the final channel balances are paid out on chain.
- **Contract Details**: This includes the merchant's Tezos public key, the on-chain ID of the contract, and the level at which the contract was originated.

## Operations

The database has an operation to retrieve each field above, given a label. Operations that mutate the database are described in this section.

All inputs are required except where noted.

| Name                                             | Input                        | Summary |
| ------------------------------------------------ | -----------------------------| --------|
| [**Insert Channel**][insert_channel]             | Label, address, `Inactive`, contract details| Insert the new channel if it doesn't already exist. If the insertion fails, return the `Inactive`. The contract details should only include the merchant's Tezos public key, not a contract ID or level.|
| [**Update Channel Status**][update_channel_status] | Label, old status, new status | Update an existing channel's status atomically.                                                    |
| **Update Closing Balances** | Label, merchant balance, customer balance (optional) | Update the closing balances to the specified values.
| **Initialize Contract Details** | Label, contract ID, level | Update the contract details. This can be called at most once.



[insert_channel]: #insert-channel
[get_address]: #get-address
[relabel_channel]: #relabel-channel
[update_channel_status]: #update-channel-status

### Insert Channel

`zkAbacus.Establish` calls this operation. This operation is atomic to prevent
race conditions. For example, if multiple Establish sessions try to insert a
channel with the same label, only the first session succeeds. Subsequent inserts
fail because a channel already exists with the given label.

### Update Channel Status

The customer uses this operation to progress the channel from one status to the
next. 
Each status transition is accompanied by a condition; the update function must be atomic and succeed if and only if the condition is met.

In a correct zkChannels execution, the statuses will transition in an expected order, based on the outcome of `zkAbacus` and `zkEscrowAgent` subprotocol executions. Establishing a channel includes the following status transitions:
```
Inactive -> Originated -> CustomerFunded -> MerchantFunded -> Ready 
```
A payment on a channel goes through a status transition loop:
```
Ready -> Started -> Locked 
   ^                  |
   |__________________|
```

The closing procedure can begin from many statuses, but in a normal run, we expect closing to start from the `Ready` status. There are no loops in close; the status progression can only move forward.
```
   _______________________________________/-> PendingMutualClose 
  |                                                              \
  |             __________________________________________________\ 
  |            |               \                                   \
Ready -> PendingExpiry -> PendingClose -> PendingCustomerClaim -> Closed
  |_______________________/     |                                  /
                                |-> Dispute_______________________/
  
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
| `PendingMutualClose` | The customer initiated a mutual close.
| `PendingExpiry`  | Merchant initiated a channel close.
| `PendingClose`   | Customer posted closing balances on chain.
| `Dispute`        | Merchant provided evidence that customer posted invalid closing balances.
| `PendingCustomerClaim` | The customer initiated a claim of the funds they are due.
| `Closed`         | The balances on the smart contract are paid out. 

### Update closing balances
This operation updates the balances of the channel as they are paid out. It enforces the following invariants:
- The merchant can receive money either one or two times.
- The merchant's balance cannot decrease in a subsequent update. 
- The customer can receive money exactly once.


## Schema

**channels**

| field                    | properties       | type                                                                                   |
| ------------------------ | ---------------- | -------------------------------------------------------------------------------------- |
| label                    | unique, required | [ChannelName][channel_name]                                                            |
| address                  | required         | [ZkChannelAddress][zk_channel_address]                                                 |
| customer_deposit | required         | [CustomerBalance][customer_balance]                                                    |
| merchant_deposit | required         | [MerchantBalance][merchant_balance]                                                    |
| state                    | required         | [ChannelState][channel_state], which is one of the options listed above |
| closing_balances | required | ClosingBalances
| merchant_tezos_public_key | required | TezosPublicKey
| contract_id | | ContractId
| level | | Level

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
