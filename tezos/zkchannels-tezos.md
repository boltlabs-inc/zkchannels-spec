# Tezos instantiation of zkChannels

The zkchannels protocol has three phases: establish, pay, and close.

# Table of Contents

  * [Establish](#establish)
    * [Open](#open)
    * [Init](#init)
    * [Contract Origination](#contract-origination)
        * [1. Requirements](#1.-requirements)
        * [2. Customer forges and signs operation](#2.-customer-forges-and-signs-operation)
        * [3. Customer injects origination operation](#3.-customer-injects-origination-operation)
        * [4. Origination confirmed ](#4.-origination-confirmed )
        * [5. Customer funds their side of the contract](#5.-customer-funds-their-side-of-the-contract)
        * [6. Merchant verifies the contract](#6.-Merchant-verifies-the-contract)

    * [Activate](#activate)
    * [Unlink](#unlink)

  * [Pay](#pay)
  * [Close](#close)
  
## Establish
(XX: TODO - add overall summary about messages sent during Establish. Something similar to 'Channel Establishment' section in LN spec)

        +-------+                              +-------+
        |       |--(1)-------  open  --------->|       |
        |       |<-(2)-------  open  ----------|       |
        |       |                              |       |
        |   A   |--(3)-------- init ---------->|   B   |
        |       |<-(4)-------- init -----------|       |
        |       |                              |       |
        |       |--(5)-- funding_confirmed --->|       |
        |       |<-(6)--- funding_locked ------|       |
        +-------+                              +-------+

        - where node A is the 'customer' and node B is the 'merchant'

## Open
(XX: We might want to add a field to the 'Open' messages to communicate `minimum_depth`, the minimum number of confirmations required for considering an operation as finalized. It is referenced below)

## Init

## Contract Origination

### 1. Requirements
Before the originating the zkChannel contract, the customer and merchant must have agreed upon the initial storage arguments, and the customer must have a closing signature from the merchant on the initial state. (XX - we could make the initial closing sig only required before funding as opposed to origination)

### Initial storage arguments
* Merchant's fixed arguments
    * [`address`:`merchAddr`]
    * [`key`:`merchPk`]
    * [`bls12_381_g2`:`g2`]
    * [`bls12_381_g2`:`merchPk0`]
    * [`bls12_381_g2`:`merchPk1`]
    * [`bls12_381_g2`:`merchPk2`]
    * [`bls12_381_g2`:`merchPk3`]
    * [`bls12_381_g2`:`merchPk4`]
    * [`bls12_381_g2`:`merchPk5`]
    * [`bls12_381_fr`:`hashCloseB`]
    * [`int`:`selfDelay`] (XX: `selfDelay` could also be channel specific)
* Channel specific arguments
    * [`bls12_381_fr`:`chanID`]
    * [`address`:`custAddr`]
    * [`key`:`custPk`]
    * [`mutez`:`custFunding`]
    * [`mutez`:`merchFunding`]

(XX: Darius TODO - Add descriptions of each variable)

#### Initial closing signature
An initial closing signature from the merchant. This is a signature on the initial channel state, which is a tuple containing the channel ID, close tag, revocation lock, cust initial balance, and merch initial balance.


### 2. Customer forges and signs operation
The customer will request their local node or tezos client to forge and sign the operation with the following inputs: (XX: Technically anyone could originate the contract on behalf of the customer. For now at least, this is written assuming the customer will.)

(XX: Darius TODO - change to tezos specific terminology)
* zkchannels contract code
* initial storage arguments
* blockchain details
* recent block header (from within the last 60 blocks)
* fees 
* tezos account public key of the sender

The fees should be handled by the customer's tezos client to optimize for cost and timeliness of the operation's confirmation.

The customer's tezos client should return a serialized and signed operation, along with it's operation hash.

### 3. Customer injects origination operation
When the customer injects their operation, they must begin watching the blockchain to ensure that the operation is confirmed. If the operation is not confirmed within 60 blocks of the block header referenced in the operation, the operation will be dropped from the mempool and the customer must go back to the previous step to forge and sign a new operation.

The customer must wait until the specified number of confirmations (XX: currently this isn't defined. In LN, before funding the channel, the non-funding party specifies `minimum_depth` to specify how many confirmations they require. This should probably happen for us during 'open' or 'init'.)

### 4. Origination confirmed 
Once the operation has reached the minimum number of required confirmations, the `contract_address` is locked in, the customer is ready to fund the contract with their initial balance.
(XX: We could decide at this point for the customer to send the merchant a `contract_originated` message with the `contract_address` and `chanID`. This may speed things up for a dual-funded channel, if the merchant was willing to fund their side of the channel before the customer funds theirs.)

### 5. Customer funds their side of the contract
The customer funds their side of the contract using the `@addFunding` entrypoint of the contract. The source of this transfer operation must be equal to the `custAddr` specified in the contract's initial storage, with the transfer amount being exactly equal to `custFunding`. 

Once the funding has been confirmed, the customer sends the merchant a `funding_confirmed` message containing the `contract_address` and `chanID`. This is to inform the merchant that the channel is ready, either for the merchant to fund their side, or if single-funded, to consider the channel open. (XX: RE `contract_address` could be sent earlier)

### 6. Merchant verifies the contract
When the merchant receives the `funding_confirmed` message they must:
* Search for the `contract_address` on chain.
* Check the contract code against their own copy of the zkchannels contract (zkchannels.tz).
* Check the contract storage. Check that for the included `chanID`, the initial storage (same as the list above: initial balances, tezos addresses, initial balances etc) are what is expected. 
* Check that the customer has fully funded their side of the channel.

If any of the above checks fail, the merchant aborts.

If it is a dual-funded channel, the merchant funds their side of the channel using the `@addFunding` entrypoint and waits for that operation to confirm. The source of this transfer operation must be equal to the `merchAddr` specified in the contract's initial storage, with the transfer amount being exactly equal to `merchFunding`. 

At this point, the merchant should check the contract storage and ensure that `status` is equal to `1`, meaning the funding is locked in. Once this contract status has reached `minimum_depth` confirmations, the merchant should send the customer a `funding_locked` message to acknowledge that the channel has been fully funded. 

(XX: A general point to do with messaging. So far I have assumed the merchant will be able to initiate communication with a specific customer. We'd need to make sure this is compatible with how zeekoe handles messaging in a privacy perserving way for the customer.)

(XX: Darius TODO - describe when `@reclaimFunding` should be used)

## Activate

## Unlink

# Pay

# Close