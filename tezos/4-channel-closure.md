# Channel Closure
  * [Mutual Close](#mutual-close)
    * [The `mutual_close_c` Message](#the-`mutual_close_c`-Message)
    * [The `mutual_close_m` Message](#the-`mutual_close_m`-message)
    * [The `@mutualClose` entrypoint](#the-`@mutualClose`-entrypoint)
  * [Unilateral Customer Close](#unilateral-customer-close)
    * [Merchant Dispute](#merchant-dispute)
  * [Unilateral Merchant Close](#unilateral-merchant-close)

A zkChannel can close either via a mutual close, in which both the customer and merchant coordinate to close the channel, or via a unilateral close, in which either the merchant or customer may initiate closing. 

## Mutual Close
Mutual close is initiated by the customer and consists of a single round of communication between the customer and the merchant, followed by a single transaction sent by the customer to the network. 

The customer sends the message `mutual_close_c` and receives back `mutual_close_m` from the merchant. Once the customer receives a valid `mutual_close_m` message, they can close the smart contract by calling the `@mutualClose` entrypoint. 

### The `mutual_close_c` Message

1. type: (`mutual_close_c`)
2. data: 
    * [`bls12_381_fr`:`cid`]
    * [`mutez`:`bal_cust`]
    * [`mutez`:`bal_merch`]
    * [`bls12_381_fr`:`rev_lock`]
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Requirements

For the customer:
  - the contents of `mutual_close_c` correspond to the latest closing state of the `zkAbacus` channel.
  - no more payments are initiated after `mutual_close_c` has been sent.

Upon receipt, the merchant:
  - checks that `rev_lock` is not in their revocation database `revocation_DB`.
  - checks that `closing_signature` is a valid signature on the proposed closing state `(cid, close, rev_lock, bal_cust, bal_merch)` with respect to the merchant's blind signing public key `merch_PS_pk`.
  - writes `rev_lock` to the database `revocation_DB`.
  - if any of these checks fail, the merchant aborts.

### The `mutual_close_m` Message

1. type: (`mutual_close_m`)
2. data: 
    * [`signature`:`mutual_close_signature`]

Here, `mutual_close_signature` is an EdDSA signature generated using `merch_pk`. The signature is over a tuple `(cid, customer_balance, merchant_balance, contract-id, "zkChannels mutual close")`, where `contract-id` is the address of the smart contract, and `context-string` is a [global default](1-setup.md#global-defaults) set to `"zkChannels mutual close"`.

#### Requirements

Upon receipt, the customer:
  - verifies `mutual_close_signature` is a valid signature with respect to `merch_pk`.
  - if the signature is not valid, aborts and initiates a [unilateral customer close](##unilateral-customer-close) 

### The `@mutualClose` entrypoint
This entry point is called with the following arguments:
* [`bls12_381_fr`:`cid`]
* [`mutez`:`bal_cust`]
* [`mutez`:`bal_merch`]
* [`signature`:`mutual_close_signature`]

#### Requirements
The `@expiry` entrypoint will only succeed if the sender is `cust_addr`, as defined in the smart contract.

## Unilateral Customer Close
For the customer to initiate a unilateral channel closure, they call the smart contract via `@custClose` with the latest state and the merchant's `closing_signature` (`s1` and `s2`) on it. Note that an operation calling the `@custClose` entrypoint will only be successful if the sender is `cust_addr`, as defined in the smart contract. The following arguments are passed into `@custClose`:
* [`bls12_381_fr`:`cid`]
* [`mutez`:`bal_cust`]
* [`mutez`:`bal_merch`]
* [`bls12_381_fr`:`rev_lock`]
* [`bls12_381_g1`:`s1`]
* [`bls12_381_g1`:`s2`]

When the `@custClose` entrypoint has been called, the merchant's balace (`bal_merch`) will automatically be transferred to the `merch_addr` by the smart contract. The customer's balance (`bal_cust`) begins a timeout period to allow the merchant to [dispute](#merchant-dispute) the latest balance. The length of the timeout period is determined by `self_delay` set in the smart contract at origination.

If the merchant does not call `@merchDispute` within the timeout period, the customer will claim their balance by calling the `@custClaim` entrypoint, at which point the smart contract with transfer the customer's balance to `cust_addr`.

### Merchant Dispute
As soon as the merchant detects that the customer has called the `@custClose` entrypoint, `rev_lock` is read from the contract storage and checked to see if it corresponds to a known `rev_secret` where H(`rev_secret`) == `rev_lock`, using the hash function SHA3-256. If `rev_secret` is known, this means the customer is attempting to close on an outdated state and the merchant will call the `@merchDispute` entrypoint with:
* [`bytes`:`rev_secret`]

If `rev_secret` hashes to `rev_lock`, the smart contract will send the customer's pending balance to the merchant. 

## Unilateral Merchant Close
As the merchant does not know the latest state of a payment channel, the merchant initiates a unilateral closure by effectively forcing the customer to close the channel within a timeout period. The length of the timeout period is determined by `self_delay` (the same as the timeout period for `@custClose`).

The merchant can initiate closure by calling the `@expiry` entrypoint. Note that an operation calling the `@expiry` entrypoint will only be successful if the sender is `merch_addr`, as defined in the smart contract.

As soon as the customer observes the `@expiry` entrypoint on the smart contract being called, they will prevent any payments on the channel from happening and call `@custClose` with the latest channel state and closing signature (as described in [Unilateral Customer Close](#unilateral-customer-close)).

If the timeout period has passed without the customer calling `@custClose`, the merchant will close the channel and receive the entire channel balance by calling the `@merchClaim` entrypoint.
