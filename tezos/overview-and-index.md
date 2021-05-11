# zkChannels on Tezos: Overview and Index


1. [System Setup and Merchant Initialization](setup.md)
3. [Channel Establishment](channel-establishment.md)
4. [Channel Payments](channel-payments.md)
5. [Channel Closure](channel-closure.md) 


## Overview
zkChannels on Tezos is built out two main components:
* zkAbacus. This component contains the functionality for a customer and a merchant to open, track payments, and collaboratively close a channel. This component does not interact with a payment network.
* Tezos-instantiated zkEscrowAgent realization. This component provides the functionality for a customer and a merchant to open and close a zkChannels escrow account as a Tezos smart contract. 

## Glossary

