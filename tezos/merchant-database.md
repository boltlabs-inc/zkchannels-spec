# Merchant Database

## Overview

A merchant in the zkChannels protocol is expected to have zkChannels open with multiple distinct customers. For each such channel, the merchant's view is limited to a channel identifier, the opening and closing balances, an on chain escrow account identifier, and the customer's Tezos public key used in the escrow account. (The escrow account is realized via [the `TezosEscrowAgent` protocol](5-tezos-escrowagent.md) as a smart contract account.) Only the associated customer has a full view of a given channel's current state.

That is, the merchant _cannot associate any given received payment to a particular channel_. Instead, the merchant keeps the information sufficient to prevent double spends and punish the customer if they attempt to close the escrow account down on an old state. Two types of information are necessary to achieve this functionality: _nonces_ and _revocation pairs_, as discussed in the [payments overview](0-overview-and-index.md#channel-payments). The merchant stores this information in a single _revocations database_ as follows.

### Revocations Database

This database consists of two independent tables:

#### Nonces

The nonces table stores a set of (non-null) nonces

#### Revocations

Each row in the revocations table must contain a revocation lock and,
optionally, a revocation secret. The merchant populates this table any time it
receives a revocation lock from the customer.

This table has the same accounting properties as an append-only ledger. Rows
can only be inserted— never updated or dropped— and there may be multiple rows
for a single revocation lock. For example, a malicious customer may first
submit a revocation lock to close a channel and then try to resubmit the lock
with its corresponding secret in a subsequent pay step. In this scenario, the
merchant stores two rows: first, the lock by itself and then the same lock with
its corresponding secret.

### Channels Database

The channels database is used to remember details about a channel across
different sessions.

- **Channel ID**: Used to look up the channel in subsequent steps to update the
  status.
- **Contract ID**: Used to perform operations on chain.
- **Merchant Deposit, Customer Deposit**: The amount initially deposited by the merchant
  and customer, respectively, upon establishing the channel. The merchant may use this for its own accounting purposes outside the scope of the protocol.
- **Status**: Used to determine valid operations on the channel.  The possible statuses are listed in the table below.
- **Closing Balances**: The balances that have been paid out on chain.

Only the status and closing balance fields can be updated.

## Operations
All inputs are required unless otherwise noted.

| Name                                                  | Input                                                                                      | Output                                                 | Summary                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------ | --------------------------------------------------- |
| [**Insert Nonce**](#insert_nonce)                      | Nonce                                                                           | True if it succeeded, False if not.                    | Insert the nonce if it doesn't already exist    |
| [**Insert Revocation Lock + Secret**](#insert_revlock) | Revocation lock, revocation secret (optional)                              | A list of previously stored pairs with the input lock. | Insert the (revocation lock, optional secret) pair  |
| [**Insert Channel**](#insert_channel)                  | Channel ID, contract ID, merchant deposit, customer deposit | None                                                   | Insert the new channel with the status "originated" |
| [**Update Channel Status**](#update_channel_status)    | Channel ID, previous status, new status                                               | None                                                   | Update an existing channel's status                 |
| [**Update Closing Balances**](#update_closing_balances) | Channel ID, merchant balance, customer balance (optional) | None | Updates an existing channel's closing balances

### Insert Nonce

`zkAbacus.Pay` calls this operation. This operation must be atomic to prevent
race conditions. For example, if multiple Pay sessions try to insert the same
nonce concurrently, only the first will succeed. Subsequent inserts will fail
because the nonce is already present.

### Insert Revocation Lock + Optional Secret

This is used in a couple situations:

1. `zkAbacus.Pay` calls this operation with the lock and secret that the
   customer reveals. If there were any previously stored pairs - in other
   words, if the merchant has previously seen this revocation lock - then the
   merchant aborts the Pay session.
2. When the merchant receives a notification that a customer has called the
   `custClose` entry point for a given channel-- either as part of a unilateral
   customer close or a unilateral merchant close-- they call this operation to
   insert the revocation lock from the customer's entrypoint call (without a
   corresponding revocation secret). The merchant then processes the output as
   specified in the [channel closing section](4-channel-closure.md):
   - _If the returned list is empty_, then the merchant has never seen the
     revocation lock before. This means the customer is closing on the most
     recent state of the channel and proceeds as specified.
   - _If the returned list contains a revocation secret_, then the customer is
     trying to resubmit old channel state. The merchant uses the secret to
     enact a punitive close by calling the `merchDispute` entrypoint as specified.
   - _If the returned list contains only locks (without any corresponding
     secrets)_, this means the customer simultaneously performed a mutual
     close while closing unilaterally, both on the most recent state. In this
     case, the merchant aborts the mutual close session.
3. When the merchant processes a mutual close request, they call this operation
   just like in an on-chain close (2). The merchant then checks the output and
   proceeds as specified in the [mutual close
   section](4-channel-closure.md#mutual-close):
   - _If the returned list is empty_, then the merchant has never seen the
     revocation lock before. This is expected, so the merchant continues the
     mutual close session as specified.
   - _If the returned list contains a revocation secret_, then the customer is
     trying to mutual-close on old channel state. The merchant aborts the
     mutual close. If the merchant knows the identity of the customer, they
     may wish to take some sort of (possibly punitive) action.
   - _If the returned list contains only locks (without any corresponding
     secrets)_, this means there is an on-chain customer close transaction on
     the most recent channel state at the time of the mutual close session.
     In this case, the merchant aborts the mutual close session as specified.

#### Optimizations

This operation may be separated into two, specialized queries if necessary. Since
Pay (1) only cares whether the set is empty or not, it could return a boolean
denoting success. Similarly, in processing an on-chain close (2) or a mutual close (3), we
could optionally return a single revocation secret if one exists.

These optimizations would minimize the rows fetched from the database and may
lead to performance improvements.

### Insert Channel

This operation is called at the end of Establish so the merchant can store the
status of the newly created channel. It stores the input Channel ID, Contract
ID, Merchant Balance, Customer Balance, and the "originated" status.

### Update Channel Status

This operation atomically verifies the current channel status and updates the channel status. There are two methods to achieve this:
- Specify both the previous and updated status. 
- Specify that the updated status is `PendingClose`. This operation will succeed only if the current status is one of `MerchantFunded`, `Active`, `PendingExpiry`, or `PendingMutualClose`.

The expected progression of a channel is:

```
Originated -> Customer funded -> Merchant funded -> Active 
```

The close procedure can be somewhat more complicated. If initiated by the merchant, the expected flow is 
```
PendingExpiry -> PendingMerchantClaim -> Closed
              \-> PendingClose ----------/ /
                               |-> Dispute / 
```
If initiated by the customer on chain, the channel will transition directly to `PendingClose` and continue as above. If initiated by the customer off chain, the channel will transition to `PendingMutualClose` to `Closed`.

For full details about when the merchant transitions from one status to the
next, [refer to the overview of the protocol](0-overview-and-index.md).

All updates must be an atomic compare-and-swap to prevent race conditions.
For example, if multiple sessions try to close the same channel concurrently, exactly one session will succeed. The other sessions will read the updated status, see that the channel is already either closed or in the process of closing, and fail.


| Status           | Condition | 
| ---------------- | --------- | 
| Originated       | The channel has a contract that is originated on chain.
| CustomerFunded   | The contract is funded by the customer, but not the merchant.
| MerchantFunded   | The contract is funded by both parties.
| Active           | The channel is funded, activated, and can process payments.
| PendingExpiry    | The merchant has initiated channel closure.
| PendingMerchantClaim | The merchant has claimed the full balance of the channel.
| PendingClose     | The customer has posted closing channel balances.
| Dispute          | The merchant has evidence that the customer posted invalid closing balances.
| PendingMutualClose | The parties have started a mutual close.
| Closed           | The channel is closed; all balances are paid out.


### Update Closing Balances
This is called when balances are finalized on chain. It maintains three invariants:
- The merchant balance can be set either once or twice.
- The merchant balance cannot decrease in a second update.
- The customer balance can be set at most once.

By definition, balances are nonnegative.


## Schema

**nonces**

| field | properties       | type      |
| ----- | ---------------- | --------- |
| data  | unique, required | [Nonce][] |

**revocations**

| field  | properties       | type                 |
| ------ | ---------------- | -------------------- |
| lock   | unique, required | [RevocationLock][]   |
| secret |                  | [RevocationSecret][] |

**channels**

| field                    | properties                                                                                                 | type                |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------- |
| channel_id               | required                                                                                                   | [ChannelId][]       |
| contract_id              | required                                                                                                   | [ContractId][]      |
| level        | required  | `Level`
| merchant_deposit | required                                                                                                   | [MerchantBalance][] |
| customer_deposit | required                                                                                                   | [CustomerBalance][] |
| status                   | required, one of the above set | [ChannelStatus][]   |
| closing_balances | required | `ClosingBalances`

[nonce]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/nonce.rs#L7-L10
[revocationlock]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/revlock.rs#L19-L22
[revocationsecret]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/revlock.rs#L24-L27
[channelid]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/states.rs#L61-L63
[contractid]: https://github.com/boltlabs-inc/zeekoe/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/src/protocol.rs#L102-L104
[merchantbalance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/states.rs#L158-L160
[customerbalance]: https://github.com/boltlabs-inc/libzkchannels-crypto/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/zkabacus-crypto/src/states.rs#L194-L196
[channelstatus]: https://github.com/boltlabs-inc/zeekoe/blob/f953b6187370f0b42edf0571c4abbae1a473e2fe/src/protocol.rs#L113-L122

#### Notes

- **required:** This column cannot be empty.
- **unique:** No two entries can contain the same value.
