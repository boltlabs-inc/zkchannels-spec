# TezosEscrowAgent 

[`TezosEscrowAgent`](5-tezos-escrowagent.md) is a realization of [the `zkEscrowAgent` functionality](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) for Tezos. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a [Tezos smart contract](2-contract-origination.md#tezos-smart-contract). 
It includes support for mutual closes, unilateral closes by either the customer or the merchant, and dispute resolution via on-chain punishment.


  - [Tezos background](#tezos-background)
    - [Operation structure](#operation-structure)
    - [Forging an operation](#forging-an-operation)
    - [Signing an operation](#signing-an-operation)
    - [Operation fees](#operation-fees)
      - [Operation weight](#operation-weight)
      - [Estimating operation gas and storage](#estimating-operation-gas-and-storage)
      - [Minimal operation fees](#minimal-operation-fees)
    - [Fee handling](#fee-handling)
  - [Tezos chain watcher requirements](#tezos-chain-watcher-requirements)
  - [Tezos client requirements](#tezos-client-requirements)
  - [zkChannels Contract](#zkchannels-contract)
    - [Initial contract arguments](#initial-contract-arguments)
    - [Global default arguments](#global-default-arguments)
    - [Fixed arguments](#fixed-arguments)
    - [Entry point requirements](#entry-point-requirements)
      - [`addCustFunding`](#addcustfunding-entrypoint)
      - [`addMerchFunding`](#addmerchfunding-entrypoint)
      - [`reclaimFunding`](#reclaimfunding-entrypoint)
      - [`expiry`](#expiry-entrypoint)
      - [`custClose`](#custclose-entrypoint)
      - [`merchDispute`](#merchdispute-entrypoint)
      - [`custClaim`](#custclaim-entrypoint)
      - [`merchClaim`](#merchclaim-entrypoint)
      - [`mutualClose`](#mutualclose-entrypoint)
  - [zkChannels Contract Origination Operation](#zkchannels-contract-origination-operation)


## Tezos background
A _Tezos operation_ is a set of instructions that transforms the state (or _context_) of the blockchain. An operation has exactly one _source account_ (the account that pays for the execution of the instructions and inclusion in the blockchain) and exactly one _destination account_ (the account that receives a transfer of funds). 

After creating (or _forging_) an operation, the operation must be signed by the private key that corresponds to the specified source account. A signed operation is then _injected_ by a Tezos node (i.e., propagated to the gossip network) for subsequent inclusion in a block. Inclusion in a block does not guarantee success of the given operation. 

An operation may be either independent or part of a larger _operation group_ (i.e., a set of operations that have been authorized by a single signature). Operations may be in the same operation group either because they have been created as a batch by the source account, or because an operation called on a contract that creates one or more operations. The operations within a given operation group have a defined order of execution and are applied to the blockchain atomically; that is, all such operations succeed or fail together. The tezos node stores an _operation status_ for each operation that indicates whether the operation is applied to the blockchain or not. 

There are two kinds of operations used in zkChannels, _origination_ operations for originating the channel's smart contract, and _transaction_ operations for funding and calling the various entrypoints of the contract.


### Operation structure

Below is an example of an unsigned Tezos operation. The signature on this operation is created by serializing the operation and signing it with the private key associated with the `source` address. For detailed description of how Tezos operations get serialized and signed see [this post](https://www.ocamlpro.com/2018/11/21/an-introduction-to-tezos-rpcs-signing-operations/).
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
- [`micheline object`:`parameters`]: Contains additional parameters used when interacting with smart contracts. The `micheline object` type is native to Tezos smart contract and is comparable to JSON. For more information [click here](https://tezos.gitlab.io/shell/micheline.html).
- [`micheline object`:`entrypoint`]: The entrypoint of the smart contract being called.
- [`micheline object`:`args`]: The arguments to be passed in to the entrypoint.

### Forging an operation
Forging is the process of creating a serialized (but unsigned) operation in Tezos. Typically this process involves interacting with a Tezos node as it requires getting recent data from the blockchain and simulating the result of the operation.

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

### Operation fees
Operation fees consist of two parts, the _baker fee_ and the _burn fee_. The baker fee is paid to the baker that bakes the block containing the given operation. This fee provides a monetary incentive for bakers to include the operation in their next block. The burn fee is removed from the total supply of tez; the burn fee amount for a given operation is determined by a protocol constant multiplied by the number of additional bytes an operation's effect adds to the state of the blockchain. 

Operation fees must be paid by the source account specified by the given operation. We have the following cases:
- If the source account has insufficient balance for the baker fee, the operation will be deemed invalid and will not be included in a block. 
- If the source account holds sufficient funds for the baker fee, but insufficient additional funds for the burn fee, the operation may be included in a block. In such a case, if the block containing the given operation is included in the blockchain, the baker receives the baker fee, but the operation itself fails.

#### Operation weight
There are two protocol constants that limit how many operations can go in a block: the first limit is on the total gas used by all operations [10,400,000](https://tzstats.com/protocols/PsFLorenaUUuikDWvMDr6fGBRG8kt3e3D3fHoXK1j1BFRxeSH4i), and the second limit is on the total size in bytes of all the operations ([512kB](https://gitlab.com/tezos/tezos/-/blob/master/src/proto_009_PsFLoren/lib_protocol/main.ml#L84) as of the Florence protocol). Given these limits, bakers are incentivized to prioritize operations that pay the highest baker fee relative to their gas and size. 

In the Tezos node reference implementation, bakers prioritize operations based on their _weight_ ([source code](https://gitlab.com/tezos/tezos/-/blob/master/src/proto_009_PsFLoren/lib_delegate/client_baking_forge.ml#L283)), where the weight is defined as:
```
weight = fee / (max ( (size/max_size), (gas/gas_block_limit)))
```
* `fee` - baker fee
* `size` - operation size in bytes
* `max_size` - size limit for all the operations in a block
* `gas` - operation gas
* `gas_block_limit` - gas limit for a block

Therefore, when creating an operation which is intended to have a fast confirmation (e.g. by the next block), the baker fee should be high enough such that the operation weight is within the baker's threshold for inclusion in the next block. It is standard practice for Tezos clients to use the [minimal operation fee](#Minimal-operation-fees).

#### Estimating operation gas and storage
The Tezos node allows clients to estimate the gas and storage used by an operation by simulating the operation locally. This lets the client know how much gas and storage is required, as well as whether the operation is successful or not (to avoid paying fees for an erroneous transaction). 

#### Minimal operation fees
Tezos nodes and bakers require a minimal operation fee to propagate operations over the gossip network and to include operations in a block. This minimal fee is not set at the protocol level but rather in the configuration of the node and the baker. Bakers may set their own minimal fee requirements that differ from the default. The default minimal fee is determined by:
```
fees >= minimal_fees + minimal_nanotez_per_byte * size + minimal_nanotez_per_gas_unit * gas
```
* `size` - the number of bytes of the complete serialized operation, including the signature.
* `gas` - operation gas

The default minimal fee values are below. For more details see the tezos [developer documentation](https://tezos.gitlab.io/protocols/004_Pt24m4xi.html).
* `minimal_fees` = 100 mutez
* `minimal_nanotez_per_gas_unit` = 100 nanotez per gas unit
* `minimal_nanotez_per_byte` = 1000 nanotez per bytes=

### Fee handling
We assume that the minimal operation fee will be sufficient for operations to be confirmed. As such, operations are forged using the minimal operation fee. This specification does not handle the case where the minimal operation fee is insufficient for bakers to include the operation into a block. 

## Tezos chain watcher requirements
The Tezos chain watcher allows a party to track the current status of a zkChannels contract. That is, the chain watcher:
* Provides details of each on-chain contract update to the party, including:
  * when an update has reached the required confirmation depth, `required_confirmations`; and
  * when an active timelock has elapsed. 
* On initialization, the chain watcher indicates the current state of the contract and whether or not that state has reached a confirmation depth of `required_confirmations`. If the current state is not confirmed to the depth `required_confirmatins`, the chain watcher will indicate when this depth has been reached.
* Must be a persistent process from the time of chain watcher initialization to the time chain watcher is halted.
* Must provide update notifications in real time, ie without delays.

## Tezos client requirements
The Tezos client is used to interact with the Tezos node for performing actions as creating operations, injecting operations and querying the state of the blockchain. We assume each party has a Tezos client capable of the following:
* Creating and injecting operations:
  * Regular transfer operations outside of the zkChannels protocol. 
  * Operations used in the zkChannels protocol.
    * Originating the zkChannels contract.
    * Calling entrypoints, including those requiring arguments.
  * Handling operation fees:
    * Given a unsigned operation, estimate the gas and storage usage and calculate the minimal operaion fee.
* Querying the blockchain:
  * Given a contract-id, return the contract code and storage.
  * Given an address, return the balance.
  * Given an operation hash, return the operation and the block height in which the given operation was included.

## zkChannels Contract 
* The contract keeps track of its current `status` that can be used to determine which entrypoint can be called. The possible contract statuses are `AWAITING_CUST_FUNDING`, `AWAITING_MERCH_FUNDING`, `OPEN`, `EXPIRY`, `CUST_CLOSE`, `CLOSED`, and `FUNDING_RECLAIMED`.
* The contract keeps track of a timeout period (denoted by `selfDelay`) that is used to determine whether an entrypoint call of type `custClaim` or `merchClaim` is legitimate.
* The contract stores `revocation_lock`, which stores the revocation lock submitted as part of a `custClose` entrypoint call as a scalar for the elliptic curve BLS12-381. 
* The contract stores the customer's closing balance after `custClose` is called.

### Representation of balances
Balances are represented as signed 64-bit integers in zkEscrowAgent. 

### Initial contract arguments
#### Channel-specific arguments
The zkChannel contract is originated with the following channel-specific arguments as specified [channel establishment](2-channel-establishment.md):
* `channel_id`: The channel identifier, a BLS12-381 scalar.
* `customer_address`: The customer's Tezos tz1 address.
* `init_customer_balance`: The customer's initial balance, converted to a signed 64-bit integer.
* `customer_public_key`: The customer's Tezos public key.
* `merchant_address`: The merchant's Tezos tz1 address.
* `init_merchant_balance`: The merchant's initial balance, converted to a signed 64-bit integer.
* `merchant_public_key`: The merchant's Tezos public key.
* `merchant_zkabacus_public_key`: The merchant's zkAbacus Pointcheval Sanders public key.

For technical reasons, an initial revocation lock is also passed into the Tezos contract origination operation:
* `initial_revocation_lock`: the BLS12-381 scalar `0x00`.

This is Tezos-specific and unlikely to be required on other chains.
#### Global default arguments
These [global default](1-setup.md#global-defaults) arguments are constant for every implementation of a zkChannels contract, regardless of the customer or merchant. 
* `close` scalar: 0x000000000000000000000000000000000000000000000000000000434c4f5345
* `context_string`: `"zkChannels mutual close"`
* `self_delay`: 172800 seconds (2 days). The time period that a party must wait to claim funds. 

The value for `close` is derived from the binary encoding of the string `'CLOSE'`. The value for `self_delay` is the number of seconds in 48 hours.

#### Fixed arguments
* `status`: An indicator that tracks the zkChannel contract's status as described in [contract requirements](#contract-requirements). This variable must be initialized to `AWAITING_CUST_FUNDING`.
* `delay_expiry`: A placeholder for the time at which funds can be claimed by a party; `delay_expiry` is of type `Timestamp` and is initialized to the Unix time `0`.

### Entrypoints

#### `addCustFunding` entrypoint
The `addCustFunding` entrypoint allows the customer to fund the contract with their initial balance, `init_customer_balance`. 
Requirements:
* The contract status must be `AWAITING_CUST_FUNDING`.
* The source must be `customer_address`.
* The amount of tez transferred to the contract must be exactly equal to `init_customer_balance`. 


On execution:
* The contract status is updated to either:
  * `AWAITING_MERCH_FUNDING` if the channel is dual-funded.
  * `OPEN` if the channel is funded by the customer alone.


#### `addMerchFunding` entrypoint
The `addMerchFunding` entrypoint allows the merchant to fund a dual-funded contract with their initial balance, `init_merchant_balance`. 
Requirements:
* The contract status must be `AWAITING_MERCH_FUNDING`.
* The source must be `merchant_address`.
* The amount of tez transferred to the contract must be exactly equal to `init_merchant_balance`.

On execution:
* The contract status is updated to `OPEN`.

#### `reclaimFunding` entrypoint
The `reclaimFunding` entrypoint allows the customer to reclaim their funding if the following conditions are met:
- The contract is for a dual-funded channel.
- The merchant has not contributed their promised funds. 

Requirements:
* The contract status must be `AWAITING_MERCH_FUNDING`.
* The source must be `customer_address`.

On execution:
* The customer's balance `customer_balance` is sent to the source of the entrypoint caller.
* The contract status is updated to `FUNDING_RECLAIMED`, which prevents any further contract activity.

#### `expiry` entrypoint
The `expiry` entrypoint allows the merchant to initiate a unilateral channel closure.

Requirements:
* The source must be `merchant_address`.
* The contract status must be `OPEN`.

On execution:
* The timestamp marking the start of the delay period (defined by `selfDelay`) is recorded by the contract.
* The contract status is updated to `EXPIRY`.

#### `custClose` entrypoint
The `custClose` entrypoint allows the customer to initate a unilateral channel closure or to responsd to an `expiry` entrypoint call.

Inputs:
* A `customer_balance` and `merchant_balance`, the closing state's balances for the customer and merchant, respectively, represented as signed 64-bit integers.
* A `revocation_lock`, the revocation lock for the closing state, a BLS12-381 scalar.
* A closing authorization signature (i.e., a signature in the zkAbacus blind signature scheme).

Requirements:
* The source must be `customer_address`.
* The contract status must be either `OPEN` or `EXPIRY`.
* The closing authorization signature must be a valid signature on the closing state that verifies under `merchant_zkabacus_public_key`. The closing state contains the `channel_id`, `close`, `revocation_lock`, `customer_balance`, and `merchant_balance`, where the customer and merchant balances are converted to BLS12-381 scalars. 

On execution:
* The customer's balance from the close state, `customer_balance`, is stored in the contract.
* The revocation lock `revocation_lock` from the close state is stored in the contract.
* The merchant's close balance, `merchant_balance`, is sent to the destination `merchant_address`.
* The timestamp marking the start of the delay period (defined by `selfDelay`) is recorded by the contract.
* The contract status is updated to `CUST_CLOSE`.

#### `merchDispute` entrypoint
The `merchDispute` entrypoint allows the merchant to dispute the closing state posted by a `custClose` entrypoint call.

Input:
* A revocation secret, `revocation_secret`, a 256-bit byte string.

Requirements:
* The source must be `merchant_address`.
* The contract status must be `CUST_CLOSE`.
* The revocation secret `revocation_secret` must satisfy `SHA3-256(revocation_secret) = revocation_lock`.

On execution:
* The customer's close balance is sent to `merchant_address`.

#### `custClaim` entrypoint
The `custClaim` entrypoint allows the customer to claim their funds after the timeout period from `custClose` has passed.

Requirements:
* The source must be `customer_address`.
* The contract status must be `CUST_CLOSE`.
* The timeout period set by `selfDelay` has passed.

On execution:
* The customer's close balance is sent to `customer_address`.
* The contract status is updated to `CLOSED`.

#### `merchClaim` entrypoint
The `merchClaim` entrypoint allows the merchant to claim their funds after the timeout period from `expiry` has passed.

Requirements:
* The source must be `merchant_address`.
* The contract status must be `EXPIRY`.
* The timeout period set by `selfDelay` has passed.

On execution:
* The entire balance is sent to `merchant_address`.
* The contract status is updated to `CLOSED`.

#### `mutualClose` entrypoint
The `mutualClose` entrypoint allows the customer to close the channel and get their balance immediately, provided they have the merchant's authorization to do so.

Inputs:
* The closing balances, `customer_balance` and `merchant_balance`, represented as signed 64-bit integers, are the amounts to be sent to `customer_address` and `merchant_address`, respectively, when the channel closes.
* A merchant EdDSA signature, `mutual_close_signature`, which is used to authorize the closing balances.

Requirements:
* The source must be `customer_address`.
* The contract status must be `OPEN`.
* `mutual_close_signature` must be a valid EdDSA signature over a byte representation of the tuple `(contract-id, context-string, channel_id, customer_balance, merchant_balance)` with respect to `merchant_public_key`. Note that while `customer_balance`, `merchant_balance`, and `mutual_close_signature` are provided to the entrypoint call as inputs, `contract-id`, `context-string`, and `channel_id` are retrieved internally from the contract's storage. `context-string` is the string  `"zkChannels mutual close"` and is defined as [global default](1-setup.md#Merchant-Setup).  The types in Michelson for this message tuple are `BLS12-381 scalar`, `string`, `address`, `mutez / i64`, and `mutez / i64`, respectively.

On execution:
* `customer_balance` and `merchant_balance` are sent to `customer_address` and `merchant_address`, respectively.
* The contract status is updated to `CLOSED`.


## zkChannels Contract Origination Operation

The origination operation contains the [zkChannels contract](#zkchannels-contract) with the following initial storage arguments:

* Merchant's fixed arguments
    * [`address`:`merchant_address`]
    * [`key`:`merchant_public_key`]
   <!--- * [`bls12_381_g2`:`g2`]
    * [`bls12_381_g2`:`X`]
    * [`bls12_381_g2`:`Y0`]
    * [`bls12_381_g2`:`Y1`]
    * [`bls12_381_g2`:`Y2`]
    * [`bls12_381_g2`:`Y3`]
    * [`bls12_381_g2`:`Y4`] 
  --->
    * [`zkabacus_key`: `merchant_zkabacus_public_key`]
    * [`domain_separator`:`close`]
    * [`int`:`self_delay`] 
    
* Channel specific arguments
    * [`ChannelID`:`channel_id`] (A BLS12-381 scalar)
    * [`address`:`customer_address`]
    * [`key`:`customer_public_key`]
    * [`mutez`:`init_customer_balance`] (A signed 64-bit integer)
    * [`mutez`:`init_merchant_balance`] (A signed 64-bit integer)

* Fixed arguments
    * [`int`:`status == AWAITING_CUST_FUNDING`]
    * [`BLS12-381_Fr`:`initial_revocation_lock`]



