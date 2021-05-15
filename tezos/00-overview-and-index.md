# zkChannels on Tezos: Overview and Index


1. [System Setup and Merchant Initialization](01-setup.md)
3. [Channel Establishment](02-channel-establishment.md)
4. [Channel Payments](03-channel-payments.md)
5. [Channel Closure](04-channel-closure.md) 


## Overview
zkChannels on Tezos is built out two main components:
* `zkAbacus`. This component contains the functionality for a customer and a merchant to open, track payments, and collaboratively close a channel. This component does not interact with a payment network.
* `TezosEscrowAgent`: A Tezos realization of the `zkEscrowAgent` functionality. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. 

## References
XX add reference to blindsigs-protocol doc.

## Glossary
* #### *`cid`*:
   * Short for 'channel identifier', a uniquely identifying hash for each channel.
* #### *`contract-id`*:
   * The smart contract's KT1 address.
* #### *inject*:
   * The process of broadcasting an operation on the Tezos blockchain.
* #### *storage*:
   * The memory held by the smart contract.