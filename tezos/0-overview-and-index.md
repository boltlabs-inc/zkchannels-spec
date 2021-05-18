# zkChannels on Tezos: Overview and Index


1. [System Setup and Merchant Initialization](1-setup.md)
2. [Channel Establishment](2-channel-establishment.md)
3. [Channel Payments](3-channel-payments.md)
4. [Channel Closure](4-channel-closure.md) 


## Overview
zkChannels on Tezos is built out two main components:
* `zkAbacus`. This component contains the functionality for a customer and a merchant to open, track payments, and collaboratively close a channel. This component does not interact with a payment network.
* `TezosEscrowAgent`: A Tezos realization of the `zkEscrowAgent` functionality. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. 

### zkChannels
zkChannels is a layer 2 protocol that enables anonymous and scalable payments between a customer (C) and a merchant (M). The customer has the ability to make payments anonymously as long as they have an open channel with sufficient balance. That is, the customer’s anonymity set for a payment is the set of all customers with whom the given merchant has a channel open.

In zkChannels, all payments must be initiated by the customer, but both positive and negative payment values are supported. 
The payment channel consists of a sequence of _states_: the customer always has a special _closing authorization signature_ for the current state when they initiate a payment with a current _payment tag_; they then receive a closing authorization signature for the new state, invalidate the old state by providing the merchant a _revocation secret_, and finally receive a payment tag for the new state. 
From the merchant’s perspective, each successful payment results in a revocation secret that allows the merchant to track whether a given state has been spent, but the merchant learns nothing else about the payment except the amount. 
The merchant is, however, confident that payments are only successful if they are initiated on a valid, unspent state of sufficient balance. 
The zkChannels protocol itself is agnostic as to the purpose of the payments, but does provide the integration points for request and provision of services, and in particular, we identify at which point of our payment subprotocol a party may reasonably consider a payment _complete_.

We briefly describe the protocol in four phases:

### (1) System Setup
Each merchant using zkChannels should generate a long-lived keypair for use
with all channels. After a merchant has completed the setup phase, a customer customer may establish a communication session with the merchant to begin setting up a new channel.

### (2) Channel initialization and establishment
Channel establishment begins with the customer sending the merchant a proposed initial state of the channel containing the initial balances. If the merchant agrees to the proposed channel, the customer and merchant can begin the initialization phase. The customer commits to the initial state, and in return gets a closing authorization signature which would allow them to close the channel on the initial state.

The customer originates the contract with the initial state and funds their side of the channel. The customer sends the newly originated contract id to the merchant, who then checks that the on-chain contract and its storage matches up with what they were expecting. If the channel is dual funded, the merchant funds their side of the channel. Once the contract is fully funded, the funds are locked in and the merchant sends the customer the first payment tag.

When the customer receives and verifies the payment tag, the channel is open and they are ready to make payments with the payments protocol.

### (3) Channel payments
Channel payments are initiated by the customer from a current state `s` and proceed as follows:
  - The customer forms and commits to a new state `s'` and proves that this state is formed from the current state using a specified payment amount `E`.
  - The customer also proves they have a valid, unspent payment tag for `s`. The merchant knows `E` and can reject the payment if the amount is unacceptable.
  - If the merchant is satisfied, the merchant blindly issues a closing authorization signature on the new state `s'`. This closing authorization signature allows the customer to initiate channel closure for the balances indicated in `s'`.
  - If the customer is satisfied the closing authorization signature is valid, the customer sends a revocation secret on `s` to the merchant. 
The revocation secret is linked to the revocation lock for `s` and allows the merchant to prove double spends and claim the entire channel balance as punishment.
  - The merchant validates the revocation lock/secret pair and, if satisfied, issues a payment tag on `s'`. This tag allows the customer to spend from `s'` in the future and completes the payment protocol.

### (4) Channel closure
There are three options for channel closure:
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