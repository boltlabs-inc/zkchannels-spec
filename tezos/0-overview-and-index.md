# zkChannels on Tezos: Overview and Index


1. [System Setup and Merchant Initialization](1-setup.md)
2. [Channel Establishment](2-channel-establishment.md)
3. [Channel Payments](3-channel-payments.md)
4. [Channel Closure](4-channel-closure.md) 


## zkChannels Overview
zkChannels is a layer 2 protocol that enables anonymous and scalable payments between a customer and a merchant. The customer has the ability to make payments anonymously as long as they have an open channel with sufficient balance. That is, the customer’s anonymity set for a payment is the set of all customers with whom the given merchant has a channel open.

zkChannels on Tezos is built out two main components:
* `zkAbacus`. This component contains the functionality for a customer and a merchant to open, track payments, and collaboratively close a channel. This component does not interact with a payment network.
* `TezosEscrowAgent`: A Tezos realization of the `zkEscrowAgent` functionality. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. 

We briefly describe the protocol in four phases:

### (1) System Setup and Merchant Initialization
Primitive specification and defaults are given [here](XX link to appropriate part of setup)
Each merchant using zkChannels generates long-lived public keys and parameters for use
with all zkChannels as detailed [here](XX link to appropriate part of setup). 


### (2) Channel establishment

The customer and merchant agree on parameters, initialize a `zkAbacus` channel, open and fund a Tezos smart contract, and activate the `zkAbacus` channel as detailed [here](XX link).

### (3) Channel payments
Channel payments are specified in (XX link design doc). The customer initiates all payment requests, but both positive and negative payment values are supported. 

In zkChannels, the customer offers a payment in return for a service from a merchant. If the merchant agrees to this payment and to provide the requested service, the parties run `zkAbacus.Pay` on the agreed-upon amount.

A zkChannel may be thought of a _sequence of _states__; the `zkAbacus` component allows state updates via `zkAbacus.Pay`:
* The customer always has a _payment tag_ for the current state, which they "spend" in `zkAbacus.Pay`.
* In return, the merchant sends the customer a _closing authorization signature_ for the new state.
* With the ability to close on the new state, the customer then invalidates the spent state's closing authorization signature by providing the merchant a _revocation secret_.
* At this point, the merchant considers the payment complete and sends a payment tag for the new state. 

From the merchant’s perspective, each successful payment results in a revocation secret that allows the merchant to track whether a given state has been invalidated, but the merchant learns nothing else about the payment except the amount. 
The merchant is, however, confident that payments are successful only if they are initiated on a valid, current state of sufficient balance. As long as `zkAbacus.Pay` completes successfully, the merchant should provide the requested service.

### (4) Channel closure (XX can you put this section in Tezos language)
There are three options for [channel closure](XX link):
  - Mutual close: The customer and merchant can collaborate off-network to create a mutual closing operation. This requires fewer on-chain operations and is therefore cheaper.
  - Unilateral customer close: The customer can unilaterally initiate channel closure by using their current closing authorization signature to create an on-network customer closing operation. The merchant's balance gets paid to them immediately. The customer's balance is held by the contract for a prespecified timeout period. If during this time, the merchant can prove it was an attempted double spend, by providing the revocation secret, the merchant can claim the customer's entire balance. After the timeout period, if the merchant has not claimed the customer's balance, the customer may claim it.
  - Unilateral merchant close: The merchant can unilaterally initiate channel closure by calling the `expiry` entrypoint on the smart contract. This operation triggers a timeout period, durinng which the customer must broadcast their latest state (as with a unilateral customer close). If the customer fails to do so within the timeout period, the merchant may claim the entire channel balance.

## References
XX add reference to blindsigs-protocol doc.

## Glossary
* **`cid`**:
   A unique channel identifier generated as a SHA3-256 hash of random contributions from both parties, together with `zkAbacus` channel parameters and `TezosEscrowAgent` escrow account parameters.
* **`contract-id`**:
   * The smart contract's KT1 address.
*  **forging**:
   The process of creating a serialized Tezos operation.
*  **inject**:
   The process of broadcasting an operation on the Tezos blockchain.
*  **storage**:
   The memory held by the smart contract.