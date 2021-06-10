# Tezos Escrow Agent Realization
* [Tezos operations background](#tezos-operations-background)
    * [Operation structure](#operation-structure)
    * [Forging an operation](#operation-structure)
    * [Signing an operation](#signing-an-operation)
* [Contract requirements](#contract-requirements)
* [Entry point requirements](#entry-point-requirements)
    * [`addFunding`](#`addfunding`)
    * [`reclaimFunding`](#`reclaimFunding`)
    * [`expiry`](#`expiry`)
    * [`custClose`](#`custClose`)
    * [`merchDispute`](#`merchDispute`)
    * [`custClaim`](#`custClaim`)
    * [`merchClaim`](#`merchClaim`)
    * [`mutualClose`](#`mutualClose`)
* [Contract Origination and Funding](#contract-origination-and-funding)
    * [Requirements](#requirements)
    * [Customer creates and signs operation](#customer-creates-and-signs-operation)
    * [Customer injects origination operation](#customer-injects-origination-operation)
    * [Origination confirmed ](#origination-confirmed )
    * [Customer funds their side of the contract](#customer-funds-their-side-of-the-contract)
    * [Merchant verifies the contract](#Merchant-verifies-the-contract)
    * [Reclaim funding](#reclaim-funding)

## Tezos operation background

Every tezos operations has only one `source` and one destination `destination`, and the operation must be signed by the private key of the `source` account. There are two `kind` of operations used in zkChannels, `origination` for originating the contract, and `transaction` for funding and calling the various entrypoints of the contract. 
### Operation structure

Below is an example of an unsigned tezos operation. The signature on this operation is created by serializing the operation and signing it with the private key associated with the `source` address. For detailed description of how Tezos operations get serialized and signed see [this post](https://www.ocamlpro.com/2018/11/21/an-introduction-to-tezos-rpcs-signing-operations/).
```
{
  "branch": "BMHBtAaUv59LipV1czwZ5iQkxEktPJDE7A9sYXPkPeRzbBasNY8",
  "contents": [
    { "kind": "transaction",
      "source": "tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51",
      "fee": "50000",
      "counter": "3",
      "gas_limit": "20000",
      "storage_limit": "0",
      "amount": "0",
      "destination": "KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa",
      "parameters": {
        "entrypoint": <entrypoint>,
        "args": <arguments>
      }
    }
}
```
- [`hash`:`branch`]: A block hash of a recent block in the Tezos blockchain. If the block hash is deeper than 60 blocks, the operation will become invalid.
- [`enum`:`kind`]: The type of operation. Origination have type `"origination"` and contract calls and regular transfers have type `"transaction"`.
- [`address`:`source`]: The sender address.
- [`mutez`:`fee`]: The fee that goes to the baker (in mutez).
- [`int64`:`counter`]: Unique sender account ‘nonce’ value. It is used to prevent replay attacks.
- [`int64`:`gas_limit`]: Caller-defined gas limit. 
- [`int64`:`storage_limit`]: Caller-defined storage limit. 
- [`mutez`:`amount`]: The value being transferred (in mutez).
- [`address`:`destination`]: The destination address.
- [`object`:`parameters`]: Contains additional parameters used when interacting with smart contracts.
- [`object`:`entrypoint`]: The entrypoint of the smart contract being called.
- [`object`:`args`]: The arguments to be passed in to the entrypoint.

### Forging an operation
Forging is the process of creating a serialized (but unsigned) operation in tezos. Typically this process involves interacting with a tezos node as it requires getting recent data from the blockchain and simulating the result of the operation.

The `fee`, `storage_limit`, `gas_limit`, and `counter` are to be determined by the user's (the customer or merchant's) Tezos node. The only requirement as far as the protocol is concerned is that the `fee` and gas and storage limits are sufficient for the operation to be baked. The `counter` is incremented each time an operation is performed from the account and is used to prevent replay attacks. Since the customer and merchant's accounts may be used to create operations outside of the zkChannels contract, the user's Tezos node should get the latest valid value from the blockchain. 

### Signing an operation
The operation is translated into a binary format, e.g.
```
ce69c5713dac3537254e7be59759cf59c15abd530d10501ccf9028a5786314cf08000002298c03ed7d454a101eb7022bc95f7e5f41ac78d0860303c8010080c2d72f0000e7670f32038107a59a2b9cfefae36ea21f5aa63c00
```
The prefix `03` is added to the bytes. `03` is a watermark for operations in Tezos.

```
03ce69c5713dac3537254e7be59759cf59c15abd530d10501ccf9028a5786314cf08000002298c03ed7d454a101eb7022bc95f7e5f41ac78d0860303c8010080c2d72f0000e7670f32038107a59a2b9cfefae36ea21f5aa63c00
```
The operation is then hashed using blake2b to create a 32-byte digest. The digest is then signed using the private key of the `source` address using the EdDSA on the Ed25519 curve.

```
637e08251cae646a42e6eb8bea86ece5256cf777c52bc474b73ec476ee1d70e84c6ba21276d41bc212e4d878615f4a31323d39959e07539bc066b84174a8ff0d
```
In the final step, the binary format of the operation is appended to the signature to create the signed operation which is ready to be broadcast by the Tezos node.





## Contract requirements

* The contract is capabale of keeping track of the balances belonging to `cust_addr` and `merch_addr`.
* The contract keeps track of its current`status` that can be used to determine which entrypoint can be called. The initial status is set to `AWAITING_FUNDING`. The possible statuses include: `AWAITING_FUNDING`, `OPEN`, `EXPIRY`, `CUST_CLOSE`, `CLOSED`.
* The contract is capable of storing `rev_lock` when submitted by `cust_addr` during `custClose`. 
* The contract is capable of keeping track of a dispute timeout period (denoted by `selfDelay`) that can be referenced when determining whether to prevent an entrypoint being called.
### Initialization arguments
The zkChannel contract is expected to be initialized with the following channel-specific arguments:
* `cid`
* `cust_addr`
* `cust_funding`
* `custPk`
* `merch_addr`
* `merchFunding`
* `merch_pk`
* `merch_PS_pk`

### Global default arguments
These [global default](1-setup.md#global-defaults) arguments are constant for every implementation of a zkChannels contract, regardless of the customer or merchant. 
* `close`
* `context_string`
* `self_delay` 


### Entry point requirements

#### `addFunding`
Allows the customer or merchant to fund their initial balance. 
Requirements:
* The entrypoint may only be called by `cust_addr` and `merch_addr`
* The status must be set to `AWAITING_FUNDING`
* The funder for `cust_funding` must be `cust_addr`
* The funder for `merch_funding` must be `merch_addr`
* The funding amount must be exactly equal to the initial balance
* The balance being funded must be 0

On execution:
* The customer or merchant's balance is updated with the funding amount
* If the customer and merchant's sides have been funded, set `status` is set to `OPEN`

#### `reclaimFunding`
Allows the customer or merchant to reclaim their funding if the other party has not funded their initial balance. 

Requirements:
* The entrypoint may only be called by `cust_addr` and `merch_addr`
* The status must be set to `AWAITING_FUNDING`
* The funding amount being claimed must have been previously funded

On execution:
* The funding amount is deducted from the customer or merchant's balance.
#### `expiry`
Allows the merchant to initiate a unilateral channel closure.

Requirements:
* The entrypoint may only be called by `merch_addr`
* The status must be set to `OPEN`

On execution:
* The delay period (defined by `selfDelay`) is triggered
* The status is set to `EXPIRY`
#### `custClose`
Allows the customer to initate a unilateral channel closure.

Inputs:
* The closing balances for the customer and merchant
* The revocation lock
* The merchant's closing authorization signature over the closing state with respect to `merch_PS_pk`

Requirements
* The entrypoint may only be called by `cust_addr`
* The status must be set to either `OPEN` or `EXPIRY`
* The `s1` component of the merchant's closing authorization signature must not be 0
* The merchant's closing authorization signature must be valid

On execution:
* The customer's close balance from the close state is stored in the contract
* `rev_lock` from the close state is stored in the contract
* The merchant's close balance is sent to `merch_addr` 
* The delay period (defined by `selfDelay`) is triggered
* The status is set to `CUST_CLOSE`

#### `merchDispute`
Allows the merchant to dispute the closing state posted during `custClose`.

Input:
* `revocation_secret`, the `SHA3-256` preimage for `rev_lock` 

Requirements:
* The entrypoint may only be called by `merch_addr`
* The status must be set to `CUST_CLOSE`
* `SHA3-256(revocation_secret)` must be equal to `rev_lock`

On execution:
* The customer's close balance is sent to `merch_addr`
#### `custClaim`
Allows the customer to claim their funds after the timeout period from `custClose` has passed.

Requirements:
* The entrypoint may only be called by `cust_addr`
* The status must be set to `CUST_CLOSE`
* The timeout period set by `selfDelay` has passed

On execution:
* The customer's close balance is sent to `cust_addr`
* The channel status is set to `CLOSED`
#### `merchClaim`
Allows the merchant to claim their funds after the timeout period from `expiry` has passed.

Requirements:
* The entrypoint may only be called by `merch_addr`
* The status must be set to `EXPIRY`
* The timeout period set by `selfDelay` has passed

On execution:
* The entire balance is sent to `merch_addr`
* The channel status is set to `CLOSED`
#### `mutualClose`
Allows the customer close the channel and get their balance immediately with the merchant's authorization.

Inputs:
* The closing balances, `cust_bal` and `merch_bal`
* The merchants signature, `merch_sig`, over the tuple (`contract-id`, `context-string`, `cid`, `cust_bal`, `merch_bal`) with respect to `merch_pk`

`<mutual_close_storage>` contains the closing balances that have been agreed upon by the customer and merchant, [`mutez`:`cust_bal`] and [`mutez`:`merch_bal`], and the merchant's EdDSA signature, [`signature`:`merch_sig`]. The expected format for `<mutual_close_storage>` is:

Requirements:
* The entrypoint may only be called by `cust_addr`
* The status must be set to `OPEN`
* `merch_sig` must be valid 

On execution:
* `cust_bal` and `merch_bal` are sent to `cust_addr` and `merch_addr` respectively
* The channel status is set to `CLOSED`


# Contract Origination and Funding
The `TezosEscrowAgent` contract origination proceeds as follows.

## Requirements
* Both the customer and merchant:
    * need tz1 accounts with a sufficient balance to carry out on-chain operations (origination, funding, and closure).
    * need online tezos clients that can create and inject operations from their (`cust_addr` and `merch_addr`).
    * have already agreed upon the initial storage arguments.
* In addition, the customer:
    * needs a closing signature from the merchant on the initial state.
### Initial storage arguments
* Merchant's fixed arguments
    * [`address`:`merch_addr`]
    * [`key`:`merch_pk`]
    * [`bls12_381_g2`:`g2`]
    * [`bls12_381_g2`:`X`]
    * [`bls12_381_g2`:`Y0`]
    * [`bls12_381_g2`:`Y1`]
    * [`bls12_381_g2`:`Y2`]
    * [`bls12_381_g2`:`Y3`]
    * [`bls12_381_g2`:`Y4`]
    * [`bls12_381_fr`:`close`]
    * [`int`:`self_delay`] 
* Channel specific arguments
    * [`bls12_381_fr`:`cid`]
    * [`address`:`cust_addr`]
    * [`key`:`custPk`]
    * [`mutez`:`custFunding`]
    * [`mutez`:`merchFunding`]

## Customer creates and signs operation
The customer will forge and sign the operation with the zkchannels contract and the initial storage arguments listed above. The operation fees are to be handled by the customer's tezos client.

## Customer broadcasts origination operation
When the customer broadcasts (or 'injects') their operation, they begin watching the blockchain to ensure that the operation is confirmed. If the operation is not confirmed within 60 blocks of the block header referenced in the operation, the operation will be dropped from the mempool and the customer must go back to the previous step to forge and sign a new operation.

## Origination confirmed 
Once the operation has reached the minimum number of required confirmations, the `contract-id` is locked in, the customer is ready to fund the contract with their initial balance.

## Customer funds their side of the contract
The customer funds their side of the contract using the `@addFunding` entrypoint of the contract. The source of this transfer operation must be equal to the `cust_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `custFunding`. 

Once the funding has been confirmed, the customer sends the merchant a `funding_confirmed` message containing the `contract-id` and `cid`. This is to inform the merchant that the channel is ready, either for the merchant to fund their side, or if single-funded, to consider the channel open. 

## Merchant verifies the contract
When the merchant receives the `open_c` message they will:
* search for the `contract-id` on chain.
* checks that the originated contract `contract-id` contains the expected zkchannels [contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz).
* check the contract storage matches the expected [initial storage](#Initial-storage-arguments)

If any of the above checks fail, the merchant aborts.

If the customer has funded their side of the channel but there are not at least `minimum_depth` confirmations, wait until there are before proceeding. 

If it is a dual-funded channel, the merchant funds their side of the channel using the `@addFunding` entrypoint and waits for that operation to confirm. The source of this transfer operation must be equal to the `merch_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `merchFunding`. 

At this point, the merchant checks the contract storage and ensure that `status` is set to `OPEN` (denoted as `1`), meaning the funding is locked in. When this status has at least `minimum_depth` confirmations, the merchant will send `open_m` to the customer.

## Reclaim funding
If during the funding process, either the customer or merchant are in the position where they have funded their own side, but the other side has not been funded, they can abort the process and reclaim their initial funds by calling the `@reclaimFunding` entrypoint. However, if both sides have funded the contract, the funds are locked in and `@reclaimFunding` will fail when called. 
