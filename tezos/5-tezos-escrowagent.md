# TezosEscrowAgent 

[`TezosEscrowAgent`](5-tezos-escrowagent.md) is a realization of [the `zkEscrowAgent` functionality](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) for Tezos. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a [Tezos smart contract](2-contract-origination.md#tezos-smart-contract). 
It includes support for mutual closes, unilateral closes by either the customer or the merchant, and dispute resolution via on-chain punishment.


- [TezosEscrowAgent](#tezosescrowagent)
  - [Tezos background](#tezos-background)
    - [Operation structure](#operation-structure)
    - [Forging an operation](#forging-an-operation)
    - [Signing an operation](#signing-an-operation)
    - [Operation fees](#operation-fees)
      - [Operation weight](#operation-weight)
      - [Fee estimation](#fee-estimation)
      - [Setting fees](#setting-fees)
      - [Minimal operation fees](#minimal-operation-fees)
  - [Contract Requirements](#contract-requirements)
    - [Initial contract arguments](#initial-contract-arguments)
    - [Global default arguments](#global-default-arguments)
    - [Fixed arguments](#fixed-arguments)
    - [Entry point requirements](#entry-point-requirements)
      - [`addFunding`](#addfunding)
      - [`reclaimFunding`](#reclaimfunding)
      - [`expiry`](#expiry)
      - [`custClose`](#custclose)
      - [`merchDispute`](#merchdispute)
      - [`custClaim`](#custclaim)
      - [`merchClaim`](#merchclaim)
      - [`mutualClose`](#mutualclose)
- [Contract Origination and Funding](#contract-origination-and-funding)
  - [Requirements](#requirements)
    - [Initial storage arguments](#initial-storage-arguments)
  - [Customer creates and signs operation](#customer-creates-and-signs-operation)
  - [Customer injects origination operation](#customer-injects-origination-operation)
  - [Origination confirmed](#origination-confirmed)
  - [Customer funds their side of the contract](#customer-funds-their-side-of-the-contract)
  - [Merchant verifies the contract](#merchant-verifies-the-contract)
  - [Reclaim funding](#reclaim-funding)

## Tezos background

Every Tezos operation has only one source and one destination, and the operation must be signed by the private key that corresponds to the source account. There are two kinds of operations used in zkChannels, origination for originating the contract, and transaction for funding and calling the various entrypoints of the contract.
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
Operation fees consist of two parts, the _baker fee_ and the _burn fee_. The baker fee is paid to the baker that creates the block containing the operation. This fee provides a monetary incentive for bakers to include the operation in their next block. The burn fee does not go to the baker is instead removed from the total supply of tez. The amount of the burn fee for an operation is determined by a protocol constant multiplied by the number of additional bytes an operation's effect adds to the state of the blockchain. 

The operation fees can only be paid for by the source account of the operation. If the source account does not hold a sufficient balance for the baker fee, it will not be an valid operation and not be included in the blockchain. If the source account holds enough for the baker fee but not enough for the burn fee as well, the baker will still receive the baker fee but the operation will fail. If an operation fails, any of its effects will be reversed (as well as any other operation within the same operation group).

#### Operation weight
As there are constraints on how many operations a block can include, bakers are incentivized to prioritize the most profitable operations for block inclusion. There are two protocol constants that limit the number of operations that can go in a block. The first is a limit on the total gas used by all operations, and the second is a limit on the total additional storage used by all operations. The total of all the operations in the block must not exceed either of these limits.

In the Tezos node reference implementation, bakers prioritize operations based on their _weight_, where the weight is defined as:
```
weight = fee / (max ( (storage/storage_block_limit), (gas/gas_block_limit)))
```
* `fee` - baker fee
* `storage` - operation storage
* `storage_block_limit` - storage limit for a block
* `gas` - operation gas
* `gas_block_limit` - gas limit for a block


#### Fee estimation
Since it is impossible to accurately predict what other operations the bakers will receive, it is also impossible to know what operation weight will be sufficient for an operation to be included in the next block. Therefore, setting the operation weight for an operation is a trade-off between cost and the probability of it being included in the next block. For a layer-2 protocol such as zkChannels, it is critical that certain operations get confirmed in a timely manner as the security of the protocol depends on the customer and merchant being able to broadcast operations within specified delay periods. We do not specify how the operation weight should be set but require that the Tezos clients used by the customer and merchant are capable of setting a target operation weight to ensure operations are confirmed quickly. 

#### Setting fees
Given a target operation weight, setting the baker fee involves two steps: estimating the gas and storage used by an operation, then setting a baker fee based on a target weight. The Tezos node allows clients to estimate the gas and storage used by an operation by simulating the operation locally. This lets the client know how much gas and storage was used, as well as whether the operation was successful or not (to avoid paying fees for an erroneous transaction). Given an estimate of the gas and storage usage, a target weight can be achieved by using the formula to derive the appropriate baker fee. 
#### Minimal operation fees
Tezos bakers by default require a minimal operation fee to propagate and include operations into a block. This minimal fee is not set at the protocol level but rather in the configuration of the node and the baker. Bakers may set their own minimal fee requirements that differ from the default. The default minimal fee is determined by:
```
fees >= minimal_fees + minimal_nanotez_per_byte * size + minimal_nanotez_per_gas_unit * gas
```
* `size` - the number of bytes of the complete serialized operation, including the signature.
* `gas` - operation gas

The default minimal fee values are below. For more details see the tezos [developer documentation](https://tezos.gitlab.io/protocols/004_Pt24m4xi.html).
* `minimal_fees` = 100 mutez
* `minimal_nanotez_per_gas_unit` = 100 nanotez per gas unit
* `minimal_nanotez_per_byte` = 1000 nanotez per bytes=




## Contract Requirements
* The contract keeps track of its current `status` that can be used to determine which entrypoint can be called. The initial status is set to `AWAITING_FUNDING`. The possible statuses are: `AWAITING_FUNDING`, `OPEN`, `EXPIRY`, `CUST_CLOSE`, `CLOSED`.
* The contract keeps track of a timeout period (denoted by `selfDelay`) that is used to determine whether an entrypoint call of type `custClaim` or `merchClaim` is legitimate.
* The contract stores `rev_lock` when submitted by `cust_addr` during `custClose`. 
* The contract stores the customer's closing balance after `custClose` is called.
### Initial contract arguments
The zkChannel contract is originated with the following channel-specific arguments as specified [channel establishment](2-channel-establishment.md):
* `cid`: The channel identifier.
* `cust_addr`: The customer's Tezos tz1 address.
* `custFunding`: The customer's initial balance.
* `custPk`: The customer's Tezos public key.
* `merch_addr`: The merchant's Tezos tz1 address.
* `merchFunding`: The merchant's initial balance.
* `merch_pk`: The merchan'ts Tezos public key.
* `merch_PS_pk`: The merchant's zkAbacus Pointcheval Sanders public key.

### Global default arguments
These [global default](1-setup.md#global-defaults) arguments are constant for every implementation of a zkChannels contract, regardless of the customer or merchant. 
* `close`: 0x000000000000000000000000000000000000000000000000000000434c4f5345
* `context_string`: `"zkChannels mutual close"`
* `self_delay`: 172800 

The value for `close` is derived from the binary encoding of the string 'CLOSE'. The value for `self_delay` is derived from the number of seconds in 48 hours.

### Fixed arguments
* `status`: A status indicator that is initialized to a value of `0` corresponding to `AWAITING_FUNDING`. This keeps track of the zkChannel contract's state, as described in [contract requirements].(#contract-requirements). 
* `rev_lock`: A placeholder for revocation locks revealed during a unilateral customer close. `rev_lock` has the type `bytes` and must be initialized with the value `0x00`.

### Entry point requirements

#### `addFunding`
The `addFunding` entrypoint allows the customer or merchant to fund their initial balance. 
Requirements:
* The entrypoint may only be called by `cust_addr` or `merch_addr`.
* The status must be set to `AWAITING_FUNDING`.
* The source for `custFunding` must be `cust_addr`.
* The source for `merchFunding` must be `merch_addr`.
* When `addFunding` is called by `cust_addr` the amount of tez being transferred to the contract must be exactly equal to `custFunding`. Similarly, when `addFunding` is called by `merch_addr` the amount of tez being transferred to the contract must be exactly equal to `merchFunding`.
* `addFunding` can only be called with `cust_addr` as a source once and `merch_addr` as a source once. 

On execution:
* The customer or merchant's balance is updated with the funding amount.
* After the entrypoint has been called, if the customer and merchant's sides are funded, the `status` is set to `OPEN`.

#### `reclaimFunding`
The `reclaimFunding` entrypoint allows the customer or merchant to reclaim their funding if the other party has not funded their initial balance. 

Requirements:
* The entrypoint may only be called by `cust_addr` or `merch_addr`.
* The status must be set to `AWAITING_FUNDING`.
* The funding amount being claimed must have been previously funded.

On execution:
* The funding amount is deducted from the customer or merchant's balance and sent to the source of the entrypoint caller.
* The status is set to `CLOSED`.
#### `expiry`
The `expiry` entrypoint allows the merchant to initiate a unilateral channel closure.

Requirements:
* The entrypoint may only be called by `merch_addr`.
* The status must be set to `OPEN`.

On execution:
* The timestamp marking the start of the delay period (defined by `selfDelay`) is recorded by the contract.
* The status is set to `EXPIRY`.
#### `custClose`
The `custClose` entrypoint allows the customer to initate a unilateral channel closure.

Inputs:
* `cust_bal` and `merch_bal`, the final balances for the customer and merchant, respectively.
* `rev_lock` revocation lock.
* The merchant's closing authorization signature over the closing state with respect to `merch_PS_pk`. The closing state contains the `cid`, `close`, `rev_lock`, `cust_bal`, and `merch_bal`. 

Requirements:
* The entrypoint may only be called by `cust_addr`.
* The status must be set to either `OPEN` or `EXPIRY`.
* The `s1` component of the merchant's closing authorization signature must not be 0.
* The merchant's closing authorization signature over the closing state must be a valid signature that verifies under `merch_PS_pk`. The closing state contains the `cid`, `close`, `rev_lock`, `cust_bal`, and `merch_bal`. 

On execution:
* The customer's balance from the close state, `cust_bal`, is stored in the contract.
* `rev_lock` from the close state is stored in the contract.
* The merchant's close balance, `merch_bal`, is sent to `merch_addr`.
* The timestamp marking the start of the delay period (defined by `selfDelay`) is recorded by the contract.
* The status is set to `CUST_CLOSE`.

#### `merchDispute`
The `merchDispute` entrypoint allows the merchant to dispute the closing state posted during `custClose`.

Input:
* `revocation_secret`, the `SHA3-256` preimage for `rev_lock` 

Requirements:
* The entrypoint may only be called by `merch_addr`.
* The status must be set to `CUST_CLOSE`.
* `SHA3-256(revocation_secret)` must be equal to `rev_lock`.

On execution:
* The customer's close balance is sent to `merch_addr`
#### `custClaim`
The `custClaim` entrypoint allows the customer to claim their funds after the timeout period from `custClose` has passed.

Requirements:
* The entrypoint may only be called by `cust_addr`.
* The status must be set to `CUST_CLOSE`.
* The timeout period set by `selfDelay` has passed.

On execution:
* The customer's close balance is sent to `cust_addr`.
* The status is set to `CLOSED`.
#### `merchClaim`
The `merchClaim` entrypoint allows the merchant to claim their funds after the timeout period from `expiry` has passed.

Requirements:
* The entrypoint may only be called by `merch_addr`.
* The status must be set to `EXPIRY`.
* The timeout period set by `selfDelay` has passed.

On execution:
* The entire balance is sent to `merch_addr`.
* The status is set to `CLOSED`.
#### `mutualClose`
The `mutualClose` entrypoint allows the customer to close the channel and get their balance immediately with the merchant's authorization.

Inputs:
* The closing balances, `cust_bal` and `merch_bal`, are the proposed amounts to be paid out to `cust_addr` and `merch_addr`, respectively, when the channel closes.
* A merchant EdDSA signature, `merch_sig`, which is used to authorize the closing balances.

Requirements:
* The entrypoint may only be called by `cust_addr`.
* The status must be set to `OPEN`.
* `merch_sig` must be valid over the tuple (`contract-id`, `context-string`, `cid`, `cust_bal`, `merch_bal`) with respect to `merch_pk`. Note that while `cust_bal`, `merch_bal`, and `merch_sig` are provided to the entrypoint call as inputs, `contract-id`, `context-string`, and `cid` are retrieved internally from the contract's storage. `context-string` is a string set to `"zkChannels mutual close"` and is defined as [global default](1-setup.md#Merchant-Setup).

On execution:
* `cust_bal` and `merch_bal` are sent to `cust_addr` and `merch_addr`, respectively.
* The status is set to `CLOSED`.


# Contract Origination and Funding
The `TezosEscrowAgent` contract origination proceeds as follows.

## Requirements
* Both the customer and merchant:
    * need tz1 accounts with a sufficient balance to carry out on-chain operations (origination, funding, and closure).
    * need online Tezos clients that can estimate fees, create, and inject operations from their (`cust_addr` and `merch_addr`).
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

* Fixed arguments
    * [`int`:`status`]
    * [`bytes`:`rev_lock`]
## Customer creates and signs operation
The customer will forge and sign the operation with the zkchannels contract and the initial storage arguments listed above. The operation fees are to be handled by the customer's Tezos client.

## Customer injects origination operation
When the customer injects the origination operation, they watch the blockchain to ensure that the operation is confirmed. If the operation is not confirmed within 60 blocks of the block header referenced in the operation, the operation will be dropped from the mempool and the customer must go back to the previous step to forge and sign a new operation.

## Origination confirmed 
Once the operation has reached the minimum number of required confirmations, the `contract-id` is locked in, the customer is ready to fund the contract with their initial balance.

## Customer funds their side of the contract
The customer funds their side of the contract using the `addFunding` entrypoint of the contract. The source of this transfer operation must be equal to the `cust_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `custFunding`. 

Once the funding has been confirmed, the customer sends the merchant a `funding_confirmed` message containing the `contract-id` and `cid`. This is to inform the merchant that the channel is ready, either for the merchant to fund their side, or if single-funded, to consider the channel open. 

## Merchant verifies the contract
When the merchant receives the `funding_confirmed` message they:
* Search for the `contract-id` on chain.
* Check that the originated contract `contract-id` contains the expected zkchannels [contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz).
* Check the contract storage matches the expected [initial storage](#Initial-storage-arguments).

If any of the above checks fail, the merchant aborts.

If the customer has funded their side of the channel but there are not at least `required_confirmations` confirmations, wait until there are before proceeding. 

If it is a dual-funded channel, the merchant funds their side of the channel using the `addFunding` entrypoint and waits for that operation to confirm. The source of this transfer operation must be equal to the `merch_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `merchFunding`. 

At this point, the merchant checks the contract storage and ensure that `status` is set to `OPEN` (denoted as `1`), meaning the funding is locked in. When this status has at least `required_confirmations` confirmations, the merchant will send `open_m` to the customer.

## Reclaim funding
If during the funding process, either the customer or merchant are in the position where they have funded their own side, but the other side has not been funded, they can abort the process and reclaim their initial funds by calling the `reclaimFunding` entrypoint. However, if both sides have funded the contract, the funds are locked in and `reclaimFunding` will fail when called. Note that once `reclaimFunding` has been called, the status will be set to `CLOSED`, preventing any further activity with the contract.
