# Contract Origination and Funding
* [Requirements](#requirements)
* [Customer forges and signs operation](#2.-customer-forges-and-signs-operation)
* [Customer injects origination operation](#3.-customer-injects-origination-operation)
* [Origination confirmed ](#4.-origination-confirmed )
* [Customer funds their side of the contract](#5.-customer-funds-their-side-of-the-contract)
* [Merchant verifies the contract](#6.-Merchant-verifies-the-contract)
* [Reclaim funding](#reclaim-funding)

## Requirements
* Both the customer and merchant:
    * need tz1 accounts with a sufficient balance to carry out on-chain operations (origination, funding, and closure).
    * need online tezos clients that can create and inject operations from their (`custAddr` and `merchAddr`).
    * have already agreed upon the initial storage arguments.
* In addition, the customer:
    * needs a closing signature from the merchant on the initial state.
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
    * [`int`:`selfDelay`] 
* Channel specific arguments
    * [`bls12_381_fr`:`cid`]
    * [`address`:`custAddr`]
    * [`key`:`custPk`]
    * [`mutez`:`custFunding`]
    * [`mutez`:`merchFunding`]

## Customer creates and signs operation
The customer will forge and sign the operation with the zkchannels contract and the initial storage arguments listed above. The operation fees are to be handled by the customer's tezos client.

## Customer broadcasts origination operation
When the customer broadcasts (or 'injects') their operation, they begin watching the blockchain to ensure that the operation is confirmed. If the operation is not confirmed within 60 blocks of the block header referenced in the operation, the operation will be dropped from the mempool and the customer must go back to the previous step to forge and sign a new operation.

The customer must wait until the specified number of confirmations `minimum_depth` to specify how many confirmations they require. This should probably happen for us during 'open' or 'init'.)

## Origination confirmed 
Once the operation has reached the minimum number of required confirmations, the `contract-id` is locked in, the customer is ready to fund the contract with their initial balance.

## Customer funds their side of the contract
The customer funds their side of the contract using the `@addFunding` entrypoint of the contract. The source of this transfer operation must be equal to the `custAddr` specified in the contract's initial storage, with the transfer amount being exactly equal to `custFunding`. 

Once the funding has been confirmed, the customer sends the merchant a `funding_confirmed` message containing the `contract_address` and `cid`. This is to inform the merchant that the channel is ready, either for the merchant to fund their side, or if single-funded, to consider the channel open. 

## Merchant verifies the contract
When the merchant receives the `funding_confirmed` message they will:
* search for the `contract_address` on chain.
* check the contract code against their own copy of the zkchannels contract (zkchannels.tz).
* check the contract storage matches the expected [initial storage](#Initial-storage-arguments)
* check that the customer has funded their side of the channel.
* the latest operation has at least `minimum_depth` confirmations.

If any of the above checks fail, the merchant aborts.

If it is a dual-funded channel, the merchant funds their side of the channel using the `@addFunding` entrypoint and waits for that operation to confirm. The source of this transfer operation must be equal to the `merchAddr` specified in the contract's initial storage, with the transfer amount being exactly equal to `merchFunding`. 

At this point, the merchant should check the contract storage and ensure that `status` is equal to `1`, meaning the funding is locked in. 

## Reclaim funding
(XX: Darius TODO - describe when `@reclaimFunding` should be used)