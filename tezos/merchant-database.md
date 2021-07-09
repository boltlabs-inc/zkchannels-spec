# Merchant Database

## Overview

A merchant in the zkChannels protocol is expected to have zkChannels open with multiple distinct customers. For each such channel, the merchant's view is limited to a channel identifier, the opening and closing balances, an on chain escrow account identifier, and the customer's Tezos public key used in the escrow account. (The escrow account is realized via [the `TezosEscrowAgent` protocol](5-tezos-escrowagent.md) as a smart contract account.) Only the associated customer has a full view of a given channel's current state.

That is, the merchant _cannot associate any given received payment to a particular channel_. Instead, the merchant keeps the information sufficient to prevent double spends and punish the customer if they attempt to close the escrow account down on an old state. Two types of information are necessary to achieve this functionality: _nonces_ and _revocation pairs_, as discussed in the [payments overview](0-overview-and-index.md#channel-payments). The merchant stores this information in a single _revocation database_ as follows.

### Revocation Database

This database consists of two independent tables: one storing a set of
(non-null) nonces, and one storing a set of revocation lock/optional-secret
pairs (the secret may be NULL but the revocation lock must be present).

### Channels

The channels database is used to remember details about a channel across
different sessions.

- **Channel ID**: Used to look up the channel in subsequent steps to update the
  status.
- **Contract ID**: Used to perform operations on chain.
- **Initial Merchant Balance**: The amount initially deposited by the merchant
  upon establishing the channel. The merchant may use this for its own
  accounting purposes, outside the scope of the protocol.
- **Initial Customer Balance**: The amount initially deposited by the customer
  upon establishing the channel. The merchant may use this for its own
  accounting purposes, outside the scope of the protocol.
- **Status**: Used to determine valid operations on the channel. For example,
  a channel with the status "originated" is not merchant funded (because it has
  not yet been customer funded). This is the only column that can be updated.
  The possible statuses are: "originated", "customer funded", "merchant
  funded", "active", "pending close", and "closed".

## Operations

| Name                                                           | Input                                                                                      | Output                                                 | Summary                                             |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------ | --------------------------------------------------- |
| [**Insert Nonce**][insert_nonce]                               | Nonce (Required)                                                                           | True if it succeeded, False if not.                    | Insert the nonce if it doesn't already exist.       |
| [**Insert Revocation Lock + Optional Secret**][insert_revlock] | Revocation Lock (Required) and a Revocation Secret (Optional)                              | A list of previously stored pairs with the input lock. | Insert the (Revocation Lock, Optional Secret) pair  |
| [**Insert Channel**][insert_channel]                           | Channel ID, Contract ID, Initial Merchant Balance, Initial Customer Balance (All Required) | None                                                   | Insert the new channel with the status "originated" |
| [**Update Channel Status**][update_channel_status]             | Channel ID (Required), New Status (Required)                                               | None                                                   | Update an existing channel's status                 |

[insert_nonce]: #insert-nonce
[insert_revlock]: #insert-revocation-lock--optional-secret
[insert_channel]: #insert-channel
[update_channel_status]: #update-channel-status

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
   just like in an on chain close (2). The merchant then checks the output and
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
     secrets)_, this means there is an on chain customer close transaction on
     the most recent channel state at the time of the mutual close session.
     In this case, the merchant aborts the mutual close session as specified.

#### Optimizations

This operation may be separated into two, specialized queries if necessary. Since
Pay (1) only cares whether the set is empty or not, it could return a boolean
denoting success. Similarly, in processing an on chain close (2) or a mutual close (3), we
could optionally return a single revocation secret if one exists.

These optimizations would minimize the rows fetched from the database and may
lead to performance improvements.

### Insert Channel

This operation is called at the end of Establish so the merchant can store the
status of the newly created channel. It stores the input Channel ID, Contract
ID, Merchant Balance, Customer Balance, and the "originated" status.

### Update Channel Status

This operation updates the channel status. The expected progression of a
channel is:

originated → customer funded → merchant funded → active → pending close → closed

In addition, any channel that's not already closed may transition to closed
when either the merchant or customer perform a unilateral close.

Any update must be an atomic compare-and-swap that asserts the previous status
to prevent race conditions. For example, if multiple sessions try to close the
same channel concurrently, only one of them will succeed. The other channel
will read the updated status, see that it's already closed, and fail.

For full details about when the merchant transitions from one status to the
next, [refer to the overview of the protocol](0-overview-and-index.md).

## Schema

**nonces**

| field | properties       |
| ----- | ---------------- |
| data  | unique, required |

**revocations**

| field  | properties       |
| ------ | ---------------- |
| lock   | unique, required |
| secret |                  |

**channels**

| field                    | properties                                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| channel_id               | required                                                                                                   |
| contract_id              | required                                                                                                   |
| initial_merchant_balance | required                                                                                                   |
| initial_customer_balance | required                                                                                                   |
| status                   | required, one of ["originated", "customer funded", "merchant funded", "active", "pending close", "closed"] |

#### Notes

- **required** - This column cannot be empty.
- **unique** - No two entries can contain the same value.
