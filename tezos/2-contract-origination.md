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
    * need online tezos clients that can create and inject operations from their (`cust_addr` and `merch_addr`).
    * have already agreed upon the initial storage arguments.
* In addition, the customer:
    * needs a closing signature from the merchant on the initial state.
### Initial storage arguments
* Merchant's fixed arguments
    * [`address`:`merch_addr`]
    * [`key`:`merch_pk`]
    * [`bls12_381_g2`:`g2`]
    * [`bls12_381_g2`:`X`]
    * [`bls12_381_g2`:`Y1`]
    * [`bls12_381_g2`:`Y2`]
    * [`bls12_381_g2`:`Y3`]
    * [`bls12_381_g2`:`Y4`]
    * [`bls12_381_g2`:`Y5`]
    * [`bls12_381_fr`:`close`]
    * [`int`:`self_delay`] 
* Channel specific arguments
    * [`bls12_381_fr`:`cid`]
    * [`address`:`cust_addr`]
    * [`key`:`custPk`]
    * [`mutez`:`custFunding`]
    * [`mutez`:`merchFunding`]

## Customer creates and signs operation
The customer will forge and sign the operation with the zkchannels contract and the initial storage arguments listed above. The operation fees are to be handled by the customer's tezos client.

## Customer broadcasts origination operation
When the customer broadcasts (or 'injects') their operation, they begin watching the blockchain to ensure that the operation is confirmed. If the operation is not confirmed within 60 blocks of the block header referenced in the operation, the operation will be dropped from the mempool and the customer must go back to the previous step to forge and sign a new operation.

## Origination confirmed 
Once the operation has reached the minimum number of required confirmations, the `contract-id` is locked in, the customer is ready to fund the contract with their initial balance.

## Customer funds their side of the contract
The customer funds their side of the contract using the `@addFunding` entrypoint of the contract. The source of this transfer operation must be equal to the `cust_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `custFunding`. 

Once the funding has been confirmed, the customer sends the merchant a `open_c` message containing the `contract-id` and `cid`. This is to inform the merchant that the channel is ready, either for the merchant to fund their side, or if single-funded, to consider the channel open. 

## Merchant verifies the contract
When the merchant receives the `open_c` message they will:
* search for the `contract-id` on chain.
* checks that the originated contract `contract-id` contains the expected zkchannels [contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz).
* check the contract storage matches the expected [initial storage](#Initial-storage-arguments)

If any of the above checks fail, the merchant aborts.

If the customer has funded their side of the channel but there are not at least `minimum_depth` confirmations, wait until there are before proceeding. 

If it is a dual-funded channel, the merchant funds their side of the channel using the `@addFunding` entrypoint and waits for that operation to confirm. The source of this transfer operation must be equal to the `merch_addr` specified in the contract's initial storage, with the transfer amount being exactly equal to `merchFunding`. 

At this point, the merchant checks the contract storage and ensure that `status` is set to `OPEN` (denoted as `1`), meaning the funding is locked in. When this status has at least `minimum_depth` confirmations, the merchant will send `open_m` to the customer.

## Reclaim funding
If during the funding process, either the customer or merchant are in the position where they have funded their own side, but the other side has not been funded, they can abort the process and reclaim their initial funds by calling the `@reclaimFunding` entrypoint. However, if both sides have funded the contract, the funds are locked in and `@reclaimFunding` will fail when called. 