# Channel Closure
  * [Mutual Close](#mutual-close)
    * [The `mutual_close_c` Message](#the-mutual_close_c-message)
    * [The `mutual_close_m` Message](#the-mutual_close_m-message)
    * [The `mutualClose` entrypoint](#the-mutualclose-entrypoint)
  * [Unilateral Customer Close](#unilateral-customer-close)
    * [Merchant Dispute](#merchant-dispute)
  * [Unilateral Merchant Close](#unilateral-merchant-close)

A zkChannel can close either via a mutual close, in which both the customer and merchant coordinate to close the channel, or via a unilateral close, in which either the merchant or customer may initiate closing. 

## Mutual Close
Mutual close is initiated by the customer and consists of a single round of communication between the customer and the merchant, followed by a single operation sent to the Tezos network.

That is:
1. The customer initiates `zkAbacus.Close` by sending the message `mutual_close_c` to the merchant.
2. The merchant runs `zkAbacus.Close`. If `zkAbacus.Close` fails, the merchant aborts. Otherwise, the merchant signs the appropriate closing message as specified below and sends this signature in the message `mutual_close_m` to the customer. 
3. The customer checks the validity of the received `mutual_close_m` message and closes the smart contract by calling [the `mutualClose` entrypoint](5-tezos-escrowagent#mutualclose-entrypoint) in the [Tezos smart contract](2-contract-origination.md#tezos-smart-contract).

The details of each message are as follows.

### The `mutual_close_c` Message

1. type: (`mutual_close_c`)
2. data: 
    * [`bls12_381_fr`:`channel_id`]
    * [`mutez`:`customer_balance`]
    * [`mutez`:`merchant_balance`]
    * [`bls12_381_fr`:`revocation_lock`]
    * [`(bls12_381_g1, bls12_381_g1)`:`closing_signature`]
      
#### Customer Requirements
The customer:
  - Updates the channel status to `PendingMutualClose` and does not initiate any more payments for the channel with identifier `channel_id`.
  - Forms the message `mutual_close_c` using the most recent closing state and closing authorization signature for the `zkAbacus` channel with identifier `channel_id`.

#### Merchant Requirements
Upon receipt, the merchant:
  - Aborts if no channel with identifier `channel_id` and status `Active` exists.
  - [Update Channel Status][merchant_update_channel_status] for the channel with identifier `channel_id` to `PendingMutualClose`.
  - Atomically [Inserts Revocation Lock][merchant_insert_revlock] if it doesn't already exist. If the value already exists in the database, the merchant aborts the mutual close session.
  - Checks that `closing_signature` is a valid Pointcheval Sanders signature on the proposed closing state `(channel_id, close, revocation_lock, customer_balance, merchant_balance)` with respect to the merchant's blind signing public key `merchant_zkabacus_public_key`. If this check fails, the merchant aborts.
 
### The `mutual_close_m` Message

1. type: (`mutual_close_m`)
2. data: [`signature`:`mutual_close_signature`]

#### Customer Requirements

Upon receipt, the customer:
  - Verifies `mutual_close_signature` is a valid EdDSA signature with respect to `merchant_public_key`.
  - If the signature is not valid, aborts and initiates a [unilateral customer close](##unilateral-customer-close).

#### Merchant Requirements
The merchant generates `mutual_close_signature` as an EdDSA signature under the merchant's Tezos public key pair (`merchant_public_key`, `merch_sk`) on the tuple `(contract-id, "zkChannels mutual close", channel_id, customer_balance, merchant_balance)`, where `contract-id` is the address of the smart contract and `context-string` is a [global default](1-setup.md#global-defaults) set to `"zkChannels mutual close"`.

### The `mutualClose` entrypoint
This entry point sends `customer_balance` funds to the customer's Tezos account (associated to `customer_public_key`) and `merchant_balance` funds to the merchant's Tezos account (associated to `merchant_public_key `).

A call to the `mutualClose` entrypoint will only succeed if and only if all of the following are true:
* The sender is `customer_address`, as defined in the smart contract.
* The contract has been funded (either by the customer for a single-funded channel, or both the customer and merchant for a dual-funded channel) and `custClose` or `expiry` have not been called.
* The signature `mutual_close_signature` is a valid EdDSA signature under the merchant's Tezos public key pair on the tuple `(contract-id, "zkChannels mutual close", channel_id, customer_balance, merchant_balance)`, where:
  * `contract-id` is the address of the smart contract;
  * `channel_id` is the channel identifier contained in the contract storage;
  * `context-string` is a [global default](1-setup.md#global-defaults) set to `"zkChannels mutual close"`;
  * the sum `customer_balance + merchant_balance` does not exceed the balance associated to the smart contract.

#### Customer Requirements
The customer calls this entrypoint with the following arguments:
* [`mutez`:`customer_balance`]
* [`mutez`:`merchant_balance`]
* [`signature`:`mutual_close_signature`]

The customer waits until this operation has reached a confirmation depth of `required_confirmations` and then updates the channel status to `Closed`.

#### Merchant Requirements
The merchant waits until this operation has reached a confirmation depth of `required_confirmations` and then [Update Channel Status][merchant_update_channel_status] to `Closed`.

## Unilateral Customer Close

The customer initiates a unilateral channel closure by updating the channel status to `PendingClose` and
calling [the `custClose` entrypoint](5-tezos-escrowagent#cust-close-entrypoint). They provide the merchant and customer balance and revocation lock from the latest state and the latest `closing_signature`. This operation only succeeds if the sender is `customer_address`, as defined in in the smart contract. 

The customer passes the following arguments to `custClose`:
* [`mutez`: `customer_balance`]
* [`mutez`: `merchant_balance`]
* [`bls12_381_fr`: `revocation_lock`]
* [`(bls12_381_g1, bls12_381_g1)`: `closing_signature`]

Calling `custClose` creates an operation group of size two: the `custClose` operation and an operation that transfers the merchant's balance from the smart contract to `merchant_address`. Either both operations will succeed or both will fail. 
A successful application timelocks the customer's balance (`customer_balance`) for a timeout period `self_delay` (set in the smart contract at origination). 

As soon as the merchant sees a `custClose` operation on chain for one of their channels, they [Update Channel Status][merchant_update_channel_status] to `PendingClose`.
The merchant reads the revocation lock `revocation_lock` from the contract storage. They call [Insert Revocation Lock][merchant_insert_revlock] with the lock to add it to their database and determine if they've previously seen a revocation secret with a SHA3-256 hash that equals `revocation_lock`.
- If so, they call [the `merchDispute` entrypoint](5-tezos-escrowagent#merchdispute-entrypoint) with the resulting revocation secret as the argument.
Calling `merchDispute` creates an operation group of size two: the `merchDispute` operation and an operation that transfers the customer's timelocked balance to the merchant's account `merchant_address`. If successful, the merchant waits until the `merchDispute` operation reaches a confirmation depth of `required_confirmations`, and then [Update Channel Status][merchant_update_channel_status] to `Closed`. They stop monitoring the contract.

    When the customer is notified that a `merchDispute` operation is posted on chain for one of their channels, they [Update Channel Status][customer_update_channel_status] to `Dispute`. 
    When they are notified that the operation reaches a confirmation depth of `required_confirmations`, they update their channel status to `Closed`.

- Otherwise, the merchant waits until the `custClose` operation is confirmed to the required confirmation depth and [Updates Channel Status][merchant_update_channel_status] to `Closed`. They stop monitoring the contract.

When the customer is notified that the timelock has elapsed, the customer updates their status from `PendingClose` to `PendingCustomerClaim` and claims their balance by calling [the `custClaim` entrypoint](5-tezos-escrowagent#custclaim).
Calling `custClaim` creates an operation group of size two: the `custClaim` operation and an operation that transfers the customer's balance from the smart contract to `customer_address`. 

When the `custClaim` operation reaches a confirmation depth of `required_confirmations`, the customer updates the channel status to `Closed`. They stop monitoring the contract.
If the operation does not succeed, the customer remains in the `PendingCustomerClaim` status.


## Unilateral Merchant Close

The merchant initiates a unilateral channel closure by calling [Update Channel Status][merchant_update_channel_status] with `PendingExpiry`. 

They call [the `expiry` entrypoint](5-tezos-escrowagent#expiry-entrypoint). This operation will only be successful if the sender is `merchant_address` as defined in the contract. 
The operation timelocks the channel balance for a timeout period of `self_delay` (set in the smart contract at origination).

As soon as the customer observes the `expiry` entrypoint on chain for one of their channels, they must complete any payment in progress. Then they decide whether to update the closing balances.
- If so, they update the channel status to `PendingClose` and call [the `custClose` entrypoint](5-tezos-escrowagent#custclose). The process continues as in [Unilateral Customer Close](#unilateral-customer-close).
- Otherwise, they update the channel status to `PendingExpiry`.

If the timeout period passes without the customer calling the `custClose` entrypoint, the merchant updates the channel status to `PendingMerchantClaim` and calls [the `merchClaim` entrypoint](5-tezos-escrowagent#merchclaim). This creates an operation group of size two: the `merchClaim` operation and an operation that transfers the entire channel balance to `merchant_address`.

Once the `merchClaim` operation has reached a confirmation depth of `required_confirmations`, the merchant calls [Update Channel Status][merchant_update_channel_status] with `Closed` and no longer needs to monitor the contract.
Similarly, the customer updates the channel status to `Closed` and no longer needs to monitor the contract.

[merchant_update_channel_status]: merchant-database.md#update-channel-status
[merchant_insert_revlock]: merchant-database.md#insert-revocation-lock--secret
[customer_update_channel_status]: customer-database.md#update-channel-status
