# zkChannels on Tezos Introduction
  * [zkChannels Overview](#zkchannels-overview) 
  * [System Setup and Merchant Initialization](#system-setup-and-merchant-initialization)
  * [Channel Establishment](#channel-establishment)
  * [Channel Payments](#channel-payments)
  * [Channel Closure](#channel-closure) 
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
* `TezosEscrowAgent`: [The TezosEscrowAgent component](5-tezos-escrowagent.md) provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. Formally, this is a realization of the [zkEscrowAgent functionality](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) for Tezos. 

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

## References
[zkChannels Private Payments Protocol](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf) <br>
[BLS12-381 Rust Crate](https://crates.io/crates/bls12_381) <br>
[libzkchannels-crypto Repository](https://github.com/boltlabs-inc/libzkchannels-crypto) <br>
[Tezos zkChannel Contract Repository](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract) <br>
[Introduction to Tezos Signing Operations](https://www.ocamlpro.com/2018/11/21/an-introduction-to-tezos-rpcs-signing-operations/)
[Micheline data format](https://tezos.gitlab.io/shell/micheline.html).
[zkChannels contract costs](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results) <br>
[Types of operation failiures](https://tezos.gitlab.io/active/michelson.html#failures)

## Glossaries and Notation
### Tezos Glossary 
*  **account**: 
   An account is a unique identifier within the protocol that is associated with a balance of tez. There are _implicit accounts_ that have addresses beginning with 'tz1', and there are _smart contract_ accounts that have addresses beginning with 'KT1'. Implicit accounts are linked to a public key and cannot include a script.
* **applied**:
   _Applied_ is the status assigned to successful operation included in the blockchain. For more details see _operation status_.
* **backtracked**:
   _Backtracked_ is the status assigned to an operation in the blockchain whose effects were reverted due to the failiure of another operation in the same operation group. A backtracked operation is one that would have otherwise been successful were it not for the failiure of a subsequent operation in the same operation group. For more details see _operation status_.
*  **baking/baker**:
   _Baking_ is the process of producing a new block in the Tezos blockchain. It is the equivalent of _mining_ on a proof of work blockchain. Baking is performed by _bakers_ who stake tezos in order to participate in the process. 
* **burn**:
   To _burn_ tez means to permanently remove it from circulation, reducing the total supply. 
* **contract identifier**:
   A smart contract's KT1 address. This acts as a unique identifier for a given contract that may be used to look up the latest state as well as any previous operations that interact with that contract.
* **confirmations**:
   _Confirmations_ represent the number of blocks in the blockchain that have been accepted by the network since the block that includes the operation of interest (including first the block it appeared in). An operation is said to have received its first confirmation when it has been included in the blockchain. Each subsequent block represents an additional confirmation.
* **destination**:
   Every operation has one destination. The destination is of an operation is the receiver's tezos address. 
* **failed**:
   _Failed_ is the status assigned to an operation in the blockchain whose effects were reverted due to an error that occured during the execution of the operation. An operation can fail for a few reasons, for example, because it encountered a programmed `FAILWITH` instruction in the smart contract, or because the source account did not have enough tez to cover the cost of the operation. For a list of reasons why an operation may fail, please refer to the tezos [developer documentation](https://tezos.gitlab.io/active/michelson.html#failures). 
*  **forging**:
   The process of creating a serialized Tezos operation.
*  **implicit account**:
   An account that is linked to a public key. Contrary to a smart contract, an implicit account cannot include a script and it cannot reject incoming transactions. Implicit accounts can use one of three curves for its signature verification. The curve being used by an account is reflected in the prefix of the account's address: 'tz1' for Ed25519, 'tz2' for Secp256k1, and 'tz3' for P256. 
*  **inject**:
   The process of broadcasting a signed operation to other Tezos nodes in the network. This must be performed by a Tezos node connected to other Tezos nodes on the network. After a successful injection, the operation will be waiting to get confirmed.
*  **KT1 address**: 
   The address of a smart contract account, which always starts with 'KT1'. The KT1 address is derived from the operation hash from which the contract was originated.
*  **mutez**:
   The smallest denomination of Tez. 1 Tez is equal to 1 million mutez.
*  **operation**:
   Operations are used for originating contracts, calling entrypoints to contracts, and transferring tez. The list of all types of operations are: origination, transaction, reveal, and delegation. _Origination_ refers to the creation of a new smart contract. _Transaction_ refers to transferring tez from one account to another. Note that contract calls are performed with _transaction_ operations where the _destination_ is the entrypoint of the contract. A _transaction_ operation can transfer a value of 0 and contain additional arguments to be passed into the entrypoint of the smart contract. Please refer to the tezos developer documentation for more information about [delegation](https://tezos.gitlab.io/introduction/howtorun.html) and [reveal](https://tezos.gitlab.io/introduction/howtouse.html#transfers-and-receipts) operations. 
*  **operation group**:
   An _operation group_ in Tezos refers to a batch of operations that have been authorized by the same signature. Operations may be in the same operation group either because they have been created as a batch by the source account, or because an operation called on a contract that created an operation. Batching operations (rather than creating separate operations) has the advantage of being more efficient in terms of gas as only one signature needs to be verified. Contract-initiated operations cannot exist independently and are always contained in the same operation group as the operation that contained the original contract call. The operations within an operation group have a defined order of execution and are applied to the blockchain atomically - either they all succeed or fail together. The tezos node stores an _operation status_ for each operation that records whether the operations in the group were applied to the blockchain or not. If the operation group failed, the operation status will indicate which operation caused the group to fail. 
*  **operations status**:
   When an operation is included in the blockchain it can either succeed or fail in terms of executing the code called on by the operation. The success of an operation is all or none; if an error occurs at any point during the execution of the operation, all the effects of the operation are reverted (apart from the baker fee being paid to the baker). Furthermore, if the operation is part of an operation group, all the operations in the group will succeed or fail together. The tezos node stores the operation status for all operation included in the blockchain. The possible operation statuses are: _applied_, _failed_, _backtracked_, and _skipped_. For a list of reasons why an operation may fail, please refer to the tezos [developer documentation](https://tezos.gitlab.io/active/michelson.html#failures).
*  **operation hash**:
   The operation hash is used as a unique identifier for operations. It is created by taking the blake2b hash over the serialized operation including the signature. 
*  **reorg**:
   A _reorg_ refers to an event where one or more blocks of the blockchain (as being observed by a particular node) is replaced with a new chain of blocks that have a higher _fitness_ score. Tezos is a proof of stake blockchain and the support for a particular block is determined by its fitness score. The fitness score of a block is a function of the amount of tezos staked by the bakers that approved the block. 
*  **skipped**:
   _Skipped_ is the status assigned to an operation in the blockchain that was skipped to the failiure of previous operation in the same operation group. An operation being skipped does not indicate whether it would or would not have been successful if it were to have been executed. For more details see _operation status_.
*  **smart contract**:
   An account which is associated with a Michelson script. They are created with an explicit origination operation and are therefore sometimes called originated accounts. The address of a smart contract always starts with the letters 'KT1'.
*  **source**:
   Every operation has one source. The source is the sender's tezos address and the operation must be signed by the source account's private key. 
*  **storage**:
   The memory held by the smart contract.
*  **tez**:
   The unit of currency in Tezo.
*  **tezos account public key** 
   Every implict account is linked to a public key. The public key is used to verify that an operation was signed by the owner of the source's address. Since an account's address is derived from the hash of a public key, it is impossible to derive the public key from the address. 
*  **tz1 address**: 
   The address of an implicit account using an EdDSA signature scheme. The address is derived from the hash of the public key and always begin with the prefix 'tz1'.

### zkChannels Glossary 
* **channel identifer**: 
   A unique identifier for a zkChannel.
* **closing state**:
   A closing state contains a channel identifier (`cid`), close flag (`close`), revocation lock (`rev_lock`), customer balance (`bal_cust`), and merchant balance (`bal_merch`). The merchant's blind signature over the closing state is what allows the customer to post the final balances to the smart contract during a unilateral customer close. The revocation lock provides a mechanism for the merchant to dispute revoked closing states posted by the customer.
* **customer**:
   A _customer_ is a user who opens a zkChannel with a merchant. The customer has a complete view of the channel state and can initiate private payments to the given merchant over their zkChannel. The customer trusts the merchant to provide a requested good or service, but does not trust the merchant with their payment history. The customer's anonymity set for payments is the set of users with whom a given merchant has an open channel. Customers are the 'spokes' in the zkChannel 'hub and spoke' network topology.
*  **timeout period**: 
   A timeout period refers to the length of time the customer or merchant has to respond to a unilateral close from the other party. In the case of a merchant unilateral close (by calling the `expiry` entrypoint), the timeout period is the time the customer has to update the contract with their latest closing state. In the case of a unilateral customer close (by calling the `custClose` entrypoint), the timeout period is the time the merchant has to dispute the closing balance by calling the `merchDispute` entrypoint. The length of the timeout period is set by `self_delay` which is defined in the [global defaults section](1-setup.md#Global-defaults).
*  **merchant**:
   A _merchant_ is an entity with the ability to accept payments and issue refunds over a zkChannel. A merchant has a limited view of channel state at any given time: they know the total amount of money allocated to a zkChannel by each participant, but cannot associate payment or refund activity to individual zkChannels. Depending on the merchant's approval process for opening channels, the merchant may or may not know the real-world identities of their customers. Merchants are the 'hubs' in the zkChannel 'hub and spoke' network topology. 
*  **revocation lock scheme**: 
   A _revocation lock scheme_ is a scheme used to prevent a customer from double spending. Each zkChannel state is associated with a _revocation lock_; when a customer makes a payment, they can revoke their previous state by revealing an associated _revocation secret_. Then if a customer attempts to make a payment on a previously revoked state, the merchant can detect and fail the requested payment. If a customer attempts to close the escrow account on a previously revoked state, the merchant may claim the customer's entire balance by providing the associated _revocation secret_ when disputing. 
*  **state**: 
   Abstractly, a zkchannel is a sequence of states, where a _state_ consists of the channel identifier, a nonce, a revocation lock, and a balance allocation.
*  **zkChannel**:
   A _zkChannel_ is a special type of payment channel that provides privacy for a customer when interacting with a merchant. More generally, a _payment channel_ allows two participants to send and receive payments to each other: when realized using cryptocurrencies, these channels typically consist of an on-chain escrow account and an off-chain state update protocol. A zkChannel is similarly comprised of two parts: an on-network escrow account and an off-network state update protocol, but the state update protocol does not allow the merchant to learn which specific channel is updated or how.

### zkChannels Notation Summary
* **`cid`**:
   A unique channel identifier generated as a SHA3-256 hash of random contributions from both parties, together with `zkAbacus` channel parameters and `TezosEscrowAgent` escrow account parameters. It is set during [channel establishment](2-channel-establishment.md) by the merchant after creating the [`open_m` message](2-channel-establishment.md#the-open_m-message), and by the customer after the receipt of the [`open_m` message](2-channel-establishment.md#the-open_m-message).
* **`close`**: 
   A fixed scalar used to differentiate closing state and state. It is defined as a [global default](1-setup.md#Global-defaults).
* **`context-string`**:
   A string set to `"zkChannels mutual close"`. This is contained in the tuple that gets signed when creating `mutual_close_signature`. This value is defined as part of the [global defaults](1-setup.md#Global-defaults).
* **`cust_pk`, `merch_pk`**:
   These refer to the customer and merchant's tezos account public keys. For the customer, it is defined during [channel establishment](2-channel-establishment.md#the-open_c-message), and for the merchant during [merchant setup](1-setup.md#Merchant-Setup).
* **`merch_PS_pk`**:
   The merchant's blind signing public key defined during [merchant setup](1-setup.md#Merchant-Setup)
* **`merch_pp_hash`**:
   This is the hash of the merchant's public parameters. It is used as a unique identifier for the merchant and is used by the customer to connect to them. `merch_pp_hash` is set to `SHA3-256(merch_PS_pk, merch_addr, merch_pk)` during the [merchant setup](1-setup.md#Merchant-Setup).
* **`required_confirmations`**:
   An integer that represents the minimum number of confirmations for an operation on the blockchain to be considered final. This value is defined as part of the [global defaults](1-setup.md#Global-defaults).
* **`rev_lock`**: 
   A revocation lock contained in a zkChannel state, generated as the SHA3-256 hash digest of a revocation secret.
* **`rev_secret`**: A revocation secret that corresponds to a revocation lock. A revocation secret is a randomly generated value.  
* **`self_delay`**: 
   This value sets the length of the timeout period during a unilateral closure where the other party must respond. The same timeout period is used for [customer initiated](4-channel-closure.md##unilateral-customer-close) unilateral closes and [merchant initiated](4-channel-closure.md##unilateral-merchant-close). The value is defined as part of the [global defaults](1-setup.md#Global-defaults).
  
