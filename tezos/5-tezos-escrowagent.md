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