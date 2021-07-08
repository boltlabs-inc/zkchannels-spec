# Merchant Database

## Overview

### Revocation Database

The merchant database `revocation_DB` is used to enforce correct `zkAbacus`
payments and `TezosEscrowAgent` account closes.

This database should consist of two independent tables: one storing a set of
(non-null) nonces, and one storing a set of revocation lock/optional-secret
pairs (the secret may be NULL but the revocation lock must be present).

### Merchant Channels

The merchant channels database is used to remember details about a channel
across different sessions.

## Operations

| Name                                                           | Input                                                                      | Output                                                 | Summary                                             |
| -------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------- |
| [**Insert Nonce**][insert_nonce]                               | Nonce (Required)                                                           | True if it succeeded, False if not.                    | Insert the nonce if it doesn't already exist.       |
| [**Insert Revocation Lock + Optional Secret**][insert_revlock] | Revocation Lock (Required) and a Revocation Secret (Optional)              | A list of previously stored pairs with the input lock. | Insert the (Revocation Lock, Optional Secret) pair  |
| [**Insert Merchant Channel**][insert_merchant_channel]         | Channel ID, Contract ID, Merchant Deposit, Customer Deposit (All Required) | None                                                   | Insert the new channel with the status "originated" |
| [**Update Channel Status**][update_channel_status]             | Channel ID (Required), New Status (Required)                               | None                                                   | Update an existing channel's status                 |

[insert_nonce]: #insert-nonce
[insert_revlock]: #insert-revocation-lock--optional-secret
[insert_merchant_channel]: #insert-merchant-channel
[update_channel_status]: #update-channel-status

### Insert Nonce

`zkAbacus.Pay` calls this operation. This operation should be atomic to prevent race conditions. For example, if
multiple Pay sessions try to insert the same nonce concurrently, only the first
should succeed. Subsequent inserts should fail because the nonce is already
present.

### Insert Revocation Lock + Optional Secret

This is used in a couple situations:

1. `zkAbacus.Pay` calls this operation with the lock and secret that the
   customer reveals. If there were any previously stored pairs - in other
   words, if the merchant has previously seen this revocation lock - then the
   merchant aborts the Pay session.
2. When the merchant receives notification that a customer has called the `custClose` entry point for a given channel, either as part of a unilateral customer close or a unilateral merchant close, they
   call this operation with the revocation lock found in the customer's entrypoint call and no revocation secret. The merchant then processes the output as specified in the [channel closing section](4-channel-closure.md#):
   - _If the returned list is empty_, then the merchant has never seen the
     revocation lock before. This means the customer is closing on the most
     recent state of the channel and proceeds as specified.
   - _If the returned list contains a revocation secret_, then the customer is
     trying to resubmit old channel state. The merchant uses the secret to
     enact a punitive close by calling the `merchDispute` entrypoint as specified.
   - _If the returned list contains only locks (without any corresponding
     secrets)_, this means the customer simultaneously performed a mutual
     close while closing unilaterally, both on the most recent state. This is
     weird behavior, but not problematic to the integrity of the protocol.
3. When the merchant processes a mutual close request, they call this operation just
   like in an on-chain close (2). The merchant then checks the
   output and proceeds as specified in the [mutual close section](4-channel-closure.md#mutual-close):
   - _If the returned list is empty_, then the merchant has never seen the
     revocation lock before. This is expected, so the merchant continues the mutual close session as specified.
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

### Insert Merchant Channel

This operation is called at the end of Establish so the merchant can remember
the state of open channels.

- **Channel ID**: Used to look up the channel in subsequent steps to update the
  state.
- **Contract ID**: Used to perform operations on-channel.
- **Merchant Deposit**: ?
- **Customer Deposit**: ?
- **Status**: Used to determine valid operations on the channel. For example,
  a channel with the status "originated" should not be merchant funded
  (because it has not yet been customer funded). This is the only column that
  can be modified.

### Update Channel Status

This operation is used to update the channel status. The expected progression
of a channel is:

originated → customer funded → merchant funded → active → closed

**TODO** Describe how channel statuses can be updated.

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

**merchant_channels**

| field            | properties                                                                                |
| ---------------- | ----------------------------------------------------------------------------------------- |
| channel_id       | required                                                                                  |
| contract_id      | required                                                                                  |
| merchant_deposit | required                                                                                  |
| customer_deposit | required                                                                                  |
| status           | required, one of ["originated", "customer funded", "merchant funded", "active", "closed"] |

**merchant_config<sup>1</sup>**

| field                            | properties |
| -------------------------------- | ---------- |
| signing_keypair                  | required   |
| revocation_commitment_parameters | required   |
| range_proof_parameters           | required   |

1 **merchant_config** will only ever have a single row corresponding to the
single merchant config.

#### Notes

- **required** - This column cannot be empty.
- **unique** - No two entries can contain the same value.
