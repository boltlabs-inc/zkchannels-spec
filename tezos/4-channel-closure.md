# Channel Closure

A zkChannel can close either via a mutual close, in which both the customer and merchant coordinate to close the channel, or via a unilateral close, in which either the merchant or customer initates the closure flow. 

## Mutual Close

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
  - the contents of `mutual_close_c` corresponds to the latest state of the payment channel.
  - no more payments are initiated after `mutual_close_c` has been sent.

Upon receipt, the merchant:
  - checks that `rev_lock` has not been seen before.
  - verifies the signature (`s1` and `s2`) against the proposed state and the merchant's PS public key.

If the merchant fails either of these checks, abort.

### The `mutual_close_m` Message

1. type: (`mutual_close_m`)
2. data: 
    * [`signature`:`mutual_close_signature`]

Where `mutual_close_signature` is an EdDSA signature on the final state using the corresponding private key of the merchant's tezos account (denoted by `merch_addr`).

#### Requirements

Upon receipt, the customer:
  - verifies the merchant's `mutual_close_signature`. 

If the signature is not valid, abort and perform a unilateral closure. 

### Customer performs mutual close
Once the customer has the merchant's signature on the latest state, they perform a mutual closure on the smart contract by calling the `@mutualClose` entrypoint with the following arguments:
* [`bls12_381_fr`:`cid`]
* [`mutez`:`bal_cust`]
* [`mutez`:`bal_merch`]
* [`bls12_381_fr`:`rev_lock`]
* [`signature`:`mutual_close_signature`]

## Unilateral Customer Close
For the customer to initial a unilateral channel closure, they call the smart contract via `@custClose` with the latest state and the merchant's `closing_signature` (`s1` and `s2`) on it:
* [`bls12_381_fr`:`cid`]
* [`mutez`:`bal_cust`]
* [`mutez`:`bal_merch`]
* [`bls12_381_fr`:`rev_lock`]
* [`bls12_381_g1`:`s1`]
* [`bls12_381_g1`:`s2`]

When the `@custClose` entrypoint has been called, the merchant's balace (`bal_merch`) will automatically be transferred to the `merch_addr` by the smart contract. The customer's balance (`bal_cust`) begins a timeout period to allow the merchant to [dispute](#merchant-dispute) the latest balance. The length of the timeout period is determined by `selfDelay` set in the smart contract at origination.

If the merchant does not call `@merchDispute` within the timeout period, the customer will claim their balance by calling the `@custClaim` entrypoint, at which point the smart contract with transfer the customer's balance to `cust_addr`.

## Merchant Dispute
As soon as the merchant detects that the customer has called the `@custClose` entrypoint, `rev_lock` is read from the contract storage and checked to see if it corresponds to a known `rev_secret` where H(`rev_secret`) == `rev_lock`, using the hash function SHA3-256. If `rev_secret` is known, this means the customer is attempting to close on an outdated state and the merchant will call the `@merchDispute` entrypoint with:
* [`bytes`:`rev_secret`]

if `rev_secret` hashes to `rev_lock`, the smart contract will send the customer's pending balance to the merchant. 

## Unilateral Merchant Close
As the merchant does not know the latest state of a payment channel, the merchant initiates a unilateral closure by effectively forcing the customer to close the channel within a timeout period. The length of the timeout period is determined by `selfDelay` (the same as the timeout period for `@custClose`).

The merchant can initiate closure by calling the `@expiry` entrypoint.

As soon as the customer observes the `@expiry` entrypoint on the smart contract being called, they will prevent any payments on the channel from happening and call `@custClose` with the latest channel state and closing signature (as described in [Unilateral Customer Close](#unilateral-customer-close)).

If the timeout period has passed without the customer calling `@custClose`, the merchant will close the channel and receive the entire channel balance by calling the `@merchClaim` entrypoint.