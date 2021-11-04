# zkChannels on Tezos Introduction
  * [zkChannels Overview](#zkchannels-overview) 
  * [System Setup and Merchant Initialization](#system-setup-and-merchant-initialization)
  * [Channel Establishment](#channel-establishment)
  * [Channel Payments](#channel-payments)
  * [Channel Closure](#channel-closure) 
  * [Implementation Considerations](#implementation-considerations)
  * [References](#references) 
  * [Glossaries and Notation](#glossaries-and-notation)
      * [Tezos Glossary](#tezos-glossary)
      * [zkChannels Glossary](#zkchannels-glossary)
      * [zkChannels Notation Summary](#zkchannels-notation-summary)



## zkChannels Overview
zkChannels is a layer 2 protocol that enables anonymous and scalable payments between a customer and a merchant. The customer has the ability to make payments anonymously as long as they have an open channel with sufficient balance. That is, the customer’s anonymity set for a payment is the set of all customers with whom the given merchant has a channel open. 

A zkChannels network for a given merchant follows a 'hub-and-spoke' topology. A hub-and-spoke topology consists of multiple outlying components (the customers), each of which is connected to a central component (the merchant).

zkChannels on Tezos is built out two main components:
* `zkAbacus`: [The zkAbacus component](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) contains the functionality for a customer and a merchant to open, track payments, and collaboratively close a channel. This component does not interact with a blockchain.
* `TezosEscrowAgent`: [The TezosEscrowAgent component](5-tezos-escrowagent.md) provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. Formally, this is a realization of the [zkEscrowAgent functionality](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) for Tezos. This realization must include [chain watcher functionality](5-tezos-escrowagent#tezos-chain-watcher-requirements) in order for channel participants to track escrow account updates and [client functionality](5-tezos-escrowagent#tezos-client-requirements) to actively interact with the chain.

We briefly describe the protocol in four phases: System Setup and Merchant Initialization, Channel Establishment, Channel Payments, and Channel Closure.

## System Setup and Merchant Initialization
Primitive specification and defaults are given [here](1-setup.md#system-setup).
Each merchant using zkChannels generates long-lived public keys and parameters for use
with all zkChannels as detailed [here](1-setup.md#merchant-setup). The merchant also initializes their [databases](merchant-database.md).


## Channel Establishment

The customer and merchant agree on parameters, initialize a `zkAbacus` channel, originate and fund a Tezos smart contract, and activate the `zkAbacus` channel as detailed [here](2-channel-establishment.md). Details specific to the smart contract functionality are given [here](5-tezos-escrowagent.md).

## Channel Payments
Channel payments are specified using zkAbacus as a building block [here](3-channel-payments.md). For full details, please see the [zkChannels Protocol document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf). 

In zkChannels, the customer offers a payment in return for a service from a merchant. If the merchant agrees to this payment and to provide the requested service, the parties run `zkAbacus.Pay` on the agreed-upon amount. The customer initiates all payment requests, but both positive and negative payment values are supported. 

A zkChannel may be thought of as a sequence of _states_; each state contains a channel identifier, the current allocation of the channel balance to customer and merchant, and information needed to track freshness of and revoke states. The `zkAbacus` component allows state updates via `zkAbacus.Pay`:
* The customer always has a _payment tag_ for the current state, which they "spend" in `zkAbacus.Pay`.
* In return, the merchant sends the customer a _closing authorization signature_ for the new state.
* With the ability to close on the new state, the customer then invalidates the spent state's closing authorization signature by providing the merchant a _revocation secret_.
* At this point, the merchant considers the payment complete and sends a payment tag for the new state. 

From the merchant’s perspective, each successful payment results in a revocation secret that allows the merchant to track whether a given state has been invalidated, but the merchant learns nothing else about the payment except the amount. 
The merchant is, however, confident that payments are successful only if they are initiated on a valid, current state of sufficient balance. As long as `zkAbacus.Pay` completes successfully, the merchant should provide the requested service.

## Channel Closure
There are three options for [channel closure](4-channel-closure.md):
  - Mutual close: The customer and merchant can collaborate off-chain to create an operation calling the `mutualClose` entrypoint in the contract. This requires fewer on-chain operations and is therefore cheaper. This procedure is built using `zkAbacus.Close` and `TezosEscrowAgent`.
  - Unilateral customer close: The customer can unilaterally initiate channel closure by using their current closing authorization signature to create an operation calling the `custClose` entrypoint. The merchant's balance gets transferred to the merchant immediately and the customer's balance is held by the contract for a prespecified timeout period. If during this time, the merchant can prove it was an attempted double spend, by providing the revocation secret, the merchant can claim the customer's entire balance. After the timeout period, if the merchant has not claimed the customer's balance, the customer may claim it via the `custClaim` entrypoint. This procedure is defined by the `TezosEscrowAgent`.
  - Unilateral merchant close: The merchant can unilaterally initiate channel closure by calling the `expiry` entrypoint on the smart contract. This operation triggers a timeout period, during which the customer must broadcast their latest state by calling `custClose` (as with a unilateral customer close). If the customer fails to do so within the timeout period, the merchant may claim the entire channel balance via the `merchClaim` entrypoint. This procedure is defined by the `TezosEscrowAgent`.

## Implementation Considerations
There are a number of implementation considerations and pitfalls in realizing the zkChannels protocol. We touch on some of them [here](implementation-considerations.md).

## References
### zkChannels Resources
[zkChannels Private Payments Protocol Document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf): Abstract zkChannels protocol specification. <br>
### zkChannels Dependencies and Implementation 
[BLS12-381 Rust Crate](https://crates.io/crates/bls12_381): Pairing-friendly curve used in zkChannels implementation. <br>
[libzkchannels-crypto](https://github.com/boltlabs-inc/libzkchannels-crypto): Implementation of cryptographic primitives used in zkChannels. <br>
[TezosEscrowAgent smart contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract): Implementation of the smart contract used in realizing the on-chain escrow functionality for zkChannels.<br>
[zkChannels contract costs](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results): Benchmarking of the Tezos smart contract.
### Tezos Resources
[Developer documentation](https://tezos.gitlab.io/index.html). At time of writing this specification, the active protocol version for Tezos is Florence. Subsections of particular interest include: 
- [Micheline](https://tezos.gitlab.io/shell/micheline.html): Data format used when passing parameters to the smart contract.
- [Michelson](https://tezos.gitlab.io/active/michelson.html): Overview of the language used for Tezos smart contracts.<br>

[An Introduction to Tezos RPCS: Signing Operations](https://www.ocamlpro.com/2018/11/21/an-introduction-to-tezos-rpcs-signing-operations/): Blog post summarizing how to forge and sign Tezos operations.<br>

## Glossaries and Notation
### Tezos Glossary 
*  **account**: 
   An _account_ is a unique identifier that is associated with a balance of tez. There are _implicit accounts_ that have addresses beginning with 'tz1', and there are _smart contract_ accounts that have addresses beginning with 'KT1'. Implicit accounts are linked to a public key and cannot include a script.
* **applied**:
   A status assigned to an operation that is included in the blockchain and is successful. For more details see _operation_ and _operation status_.
* **backtracked**:
   A status assigned to an operation that is included in the blockchain and previously successful, but the effects of which have been reverted. This occurs when a subsequent operation in the same operation group fails. For more details see _operation_, _operation group_, and _operation status_.
*  **bake**: To produce a new block in the Tezos blockchain. It is the equivalent of _mining_ on a proof-of-work blockchain.
*  **baker**: A Tezos node that _bakes_ blocks. Also known as a _validator_. 
*  **baker fee**:
   All operations contain a _baker fee_ that is paid to the baker that creates the block containing the operation. This fee provides a monetary incentive for bakers to include the operation in their next block.
* **burn**:
   To permanently remove _tez_ from circulation, thereby reducing the total supply. 
*  **burn fee**:
   A fee paid by an operation's source account for adding storage to the state of the blockchain. The amount of the burn fee for an operation is determined by a protocol constant multiplied by the number of additional bytes an operation's effect adds to the state of the blockchain. 
* **contract identifier**:
   A _smart contract's KT1 address_. This acts as a unique identifier for a given contract that may be used to look up the latest state as well as any previous operations that interact with that contract.
* **confirmation**: A _confirmation_ of a given operation is any block included in the blockchain that implies the operation in question is also included in the blockchain. That is, the first block to include the given operation or any subsequent block thereof is a confirmation.
* **confirmation depth**:
   The total number of confirmations of a given operation. The index starts at 1, e.g. an operation that is included in the latest block of the blockchain will have a confirmation depth of 1.
* **destination**: The intended receiver's tezos address for an operation; every operation has exactly one destination.  
* **failed**:
   A status assigned to an operation that is included in the blockchain, but the execution of which has failed. An operation can fail for a few reasons, e.g., due to a programmed `FAILWITH` instruction in the smart contract, or because the source account does not have sufficient tez to cover the cost of the operation. For a list of reasons why an operation may fail, see the [tezos developer documentation](https://tezos.gitlab.io/active/michelson.html#failures). 
*  **forge**:
   To create a serialized Tezos operation.
*  **gas**:
   A measure of the number of elementary operations performed during the execution of a smart contract. Gas is used to measure how much computing power is used to execute a smart contract.
*  **implicit account**:
   An account that is linked to a public key. An implicit account cannot include a script and cannot reject incoming transactions. The address prefix indicates the instantiation of EdDSA signature scheme for the associated public key, i.e. 'tz1' indicates Ed25519, 'tz2' indicates ECDSA over Secp256k1, and 'tz3' indicates ECDSA over P256. 
*  **inject**:
   To broadcast a signed operation to Tezos nodes in the network; injection is performed by a Tezos node.
*  **KT1 address**: 
   The address of a smart contract account, which always starts with 'KT1'. The KT1 address is derived from the operation hash of the contract's originating operation.
*  **mempool**:
   A node's mechanism for storing unconfirmed operations.
*  **minimal operation fee**:
   The minimal fee required by either a Tezos node in order to propagate operations over the gossip network or required by a baker in order to include an operation into a block. This fee is set in the Tezos node's configuration. For more information see the [developer documentation](https://tezos.gitlab.io/protocols/004_Pt24m4xi.html).
*  **mutez**:
   The smallest denomination of tez. One tez is equal to one million mutez.
*  **nanotez**:
   _Nanotez_ are used for fine-grained gas calculations for fees. One nanotez is equal to one thousand mutez. Fees calculated using nanotez are rounded to mutez when being defined in operations.
*  **tezos node**:
   A peer in the peer-to-peer network. A tezos node maintains local state and propagates blocks and operations in the gossip network. A tezos node may also participate in baking.
*  **operation**:
   An _operation_ is a set of instructions that transform the state of the blockchain. Supported operation types are as follows:
   - **origination**: An operation that creates a new smart contract. 
   - **transaction**: An operation that transfers tez from one account to another. Note that contract calls are performed with _transaction_ operations where the _destination_ is the entrypoint of the contract. A _transaction_ operation can transfer a value of 0 and contain additional arguments to be passed into the entrypoint of the smart contract. 
   - **delegation**: See the tezos developer documentation for more information about [delegation operations](https://tezos.gitlab.io/introduction/howtorun.html).
   - **reveal**: See the tezos developer documentation for more information about [reveal operations](https://tezos.gitlab.io/introduction/howtouse.html#transfers-and-receipts) operations. 
*  **operation group**:
   A batch of operations that have been authorized by the same signature. Operations may be in the same operation group either because they have been created as a batch by the source account, or because an operation called on a contract that created an operation. Batching operations (rather than creating separate operations) has the advantage of being more efficient in terms of gas costs, as only one signature needs to be verified. Contract-initiated operations cannot exist independently and are always contained in the same operation group as the operation that contained the original contract call. The operations within a given operation group have a defined order of execution and are applied to the blockchain atomically; that is, all such operations succeed or fail together. The tezos node stores an _operation status_ for each operation that indicates whether the operation was applied to the blockchain or not. 
*  **operation status**:
   Used to indicate the effect of an operation included in the blockchain; allowed values are _applied_, _failed_, _backtracked_, and _skipped_. That is, when an operation is included in the blockchain, the execution of the code called on by that operation can either succeed or fail. The success of an operation is all or none; if an error occurs at any point during the execution of the operation, all the effects of the operation are reverted, apart from the baker fee sent to the baker. If the operation is part of an operation group, all the operations in the group succeed or fail together. The tezos node stores the operation status for all operations included in the blockchain. For a list of reasons why an operation may fail, please refer to the tezos [developer documentation](https://tezos.gitlab.io/active/michelson.html#failures).
*  **operation hash**:
   A unique identifier for a given operation, created by taking the blake2b hash over the serialized operation, including the signature. 
*  **reorg**:
   An event where one or more blocks of the blockchain (as being observed by a particular node) are replaced with a new chain of blocks that have a higher [fitness score](https://tezos.gitlab.io/alpha/glossary.html?highlight=fitness#score). A block's fitness score is a function of the amount of tezos staked by the bakers that approved the block in question. 
*  **skipped**:
   A status assigned to an operation included the blockchain, the execution of which has been skipped due to the failure of a previous operation in the same operation group. For more details see _operation status_.
*  **smart contract**:
   An account which is associated with a Michelson script. Smart contracts are created with an explicit origination operation and are sometimes called _originated accounts_. The address of a smart contract always starts with the letters 'KT1'.
*  **source**: The intended sender's tezos address for an operation; every operation has exactly one source and must be signed by the source account's associated private key. 
*  **storage**:
   The memory held by the smart contract.
*  **tez**:
   The unit of currency in Tezo.
*  **tezos account public key**:
   Every implict account is linked to a public key. The public key is used to verify that an operation was signed by the owner of the source's address. Since an account's address is derived from the hash of a public key, it is impossible to derive the public key from the address. 
*  **tz1 address**: 
   The address of an implicit account using the Ed25519 signature scheme. The address is derived from the hash of the public key and always begin with the prefix 'tz1'.
*  **weight**:
   The default measure used in the Tezos node reference implementation for prioritizing operations for inclusion in a block. An operation's _weight_ is defined by `weight = fee / (max ( (storage/storage_block_limit), (gas/gas_block_limit)))` where `fee` is the baker fee, `storage` is the operation storage, `storage_block_limit` is the storage limit for a block, `gas` is the operation gas, and `gas_block_limit` is the gas limit for a block ([see source code](https://gitlab.com/tezos/tezos/-/blob/master/src/proto_009_PsFLoren/lib_delegate/client_baking_forge.ml#L283)).
### zkChannels Glossary 
* **channel identifer**: 
   A unique identifier for a zkChannel, represented as a BLS12-381 scalar.
* **closing state**:
   A closing state is a tuple that contains a channel identifier (`channel_id`), close flag (`close`), revocation lock (`revocation_lock`), customer balance (`customer_balance`), and merchant balance (`merchant_balance`); each element is represented as a BLS12-381 scalar. The merchant's blind signature over the closing state is what allows the customer to post the final balances to the smart contract during a unilateral customer close. The revocation lock provides a mechanism for the merchant to dispute revoked closing states posted by the customer.
* **customer**:
   A _customer_ is a user who opens a zkChannel with a merchant. The customer has a complete view of the channel state and can initiate private payments to the given merchant over their zkChannel. The customer trusts the merchant to provide a requested good or service, but does not trust the merchant with their payment history. The customer's anonymity set for payments is the set of users with whom a given merchant has an open channel. Customers are the 'spokes' in the zkChannel 'hub and spoke' network topology.
*  **timeout period**: 
   A timeout period refers to the length of time the customer or merchant has to respond to a unilateral close from the other party. In the case of a merchant unilateral close (by calling the `expiry` entrypoint), the timeout period is the time the customer has to update the contract with their latest closing state. In the case of a unilateral customer close (by calling the `custClose` entrypoint), the timeout period is the time the merchant has to dispute the closing balance by calling the `merchDispute` entrypoint. The length of the timeout period is set by `self_delay` which is defined in the [global defaults section](1-setup.md#Global-defaults).
*  **merchant**:
   A _merchant_ is an entity with the ability to accept payments and issue refunds over a zkChannel. A merchant has a limited view of channel state at any given time: they know the total amount of money allocated to a zkChannel by each participant, but cannot associate payment or refund activity to individual zkChannels. Depending on the merchant's approval process for opening channels, the merchant may or may not know the real-world identities of their customers. Merchants are the 'hubs' in the zkChannel 'hub and spoke' network topology. 
*  **revocation lock scheme**: 
   A _revocation lock scheme_ is a scheme used to prevent a customer from double spending. Each zkChannel state is associated with a _revocation lock_; when a customer makes a payment, they can revoke their previous state by revealing an associated _revocation secret_. Then if a customer attempts to make a payment on a previously revoked state, the merchant can detect and fail the requested payment. If a customer attempts to close the escrow account on a previously revoked state, the merchant may claim the customer's entire balance by providing the associated _revocation secret_ when disputing. 
*  **state**: 
   Abstractly, a zkchannel is a sequence of states, where a _state_ is a tuple of BLS12-381 scalars, consisting of the channel identifier, a nonce, a revocation lock, and a balance allocation.
*  **zkChannel**:
   A _zkChannel_ is a special type of payment channel that provides privacy for a customer when interacting with a merchant. More generally, a _payment channel_ allows two participants to send and receive payments to each other: when realized using cryptocurrencies, these channels typically consist of an on-chain escrow account and an off-chain state update protocol. A zkChannel is similarly comprised of two parts: an on-network escrow account and an off-network state update protocol, but the state update protocol does not allow the merchant to learn which specific channel is updated or how.

### zkChannels Notation Summary
* **`channel_id`**:
   A unique channel identifier generated as a SHA3-256 hash of random contributions from both parties, together with `zkAbacus` channel parameters and `TezosEscrowAgent` escrow account parameters, and then converted to a BLS12-381 scalar. It is set during [channel establishment](2-channel-establishment.md) by the merchant after creating the [`open_m` message](2-channel-establishment.md#the-open_m-message), and by the customer after the receipt of the [`open_m` message](2-channel-establishment.md#the-open_m-message).
* **`close`**: 
   A fixed scalar used to differentiate closing state and state. It is defined as a [global default](1-setup.md#Global-defaults).
* **`context-string`**:
   A string set to `"zkChannels mutual close"`. This is contained in the tuple that gets signed when creating `mutual_close_signature`. This value is defined as part of the [global defaults](1-setup.md#Global-defaults).
* **`customer_public_key`, `merchant_public_key`**:
   These refer to the customer and merchant's tezos account public keys. For the customer, it is defined during [channel establishment](2-channel-establishment.md#the-open_c-message), and for the merchant during [merchant setup](1-setup.md#Merchant-Setup).
* **`merchant_zkabacus_public_key`**:
   The merchant's blind signing public key defined during [merchant setup](1-setup.md#Merchant-Setup)
* **`merch_pp_hash`**:
   This is the hash of the merchant's public parameters. It is used as a unique identifier for the merchant and is used by the customer to connect to them. `merch_pp_hash` is set to `SHA3-256(merchant_zkabacus_public_key, merchant_address, merchant_public_key)` during the [merchant setup](1-setup.md#Merchant-Setup).
* **`required_confirmations`**:
   An integer that represents the minimum number of confirmations for an operation on the blockchain to be considered final. This value is defined as part of the [global defaults](1-setup.md#Global-defaults).
* **`revocation_lock`**: 
   A revocation lock contained in a zkChannel state, generated as the SHA3-256 hash digest of a revocation secret.
* **`revocation_secret`**: A revocation secret that corresponds to a revocation lock. A revocation secret is a randomly generated value.  
* **`self_delay`**: 
   This value sets the length of the timeout period during a unilateral closure where the other party must respond. The same timeout period is used for [customer initiated](4-channel-closure.md##unilateral-customer-close) unilateral closes and [merchant initiated](4-channel-closure.md##unilateral-merchant-close). The value is defined as part of the [global defaults](1-setup.md#Global-defaults).
  
