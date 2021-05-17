# Channel Establishment

  * [Overview](#Overview)
  * [Global defaults](#global-defaults)
  * [The `prep_c` Message](#the-`prep_c`-Message)
  * [The `prep_m` Message](#the-`prep_m`-Message)
  * [The `init_c` Message](#the-`init_c`-Message)
  * [The `init_m` Message](#the-`init_m`-Message)
  * [The `open_c` Message](#the-`open_c`-Message)
  * [The `open_m` Message](#the-`open_m`-Message)

## Prerequistes
The merchant has completed the [setup](#1-setup.md) phase, and the customer and merchant have established a communication session.

The customer has [obtained the merchant’s setup information](1-setup.md) out of band beforehand. The merchant public parameters include the following: XX add me.

## Overview
Channel establishment is a three round protocol.

In the first round, the customer sends the `prep_c` message, which contains information about the initial state of the proposed channel. If the merchant agrees to open the proposed channel, they reply with the message `prep_m`, which contains the merchan'ts contribution to the channel identifier. At the end of this round, the customer and merchant have exchanged enough information to compute the channel identifer `cid`, which acts as the unique channel identifier for the on-chain Tezos escrow account and off-chain `zkAbacus` channel.

In the next round, the customer and merchant initialize the `zkAbacus` channel by running `zkAbacus.Initialize()` on the previously established parameters. In this subroutine, the customer sends `init_c`, which consists of a (hiding) commitment to the intial state and a zero-knowledge proof of correctness. In return, the merchant sends `init_m`, which contains an initial closing authorization signature for the customer.

For the final round, the customer originates the contract and funds their side of the channel. The customer sends `open_c` to the merchant, which contains the contract id `contract-ID`. The merchant checks the corresponding contract and initial storage for correctness. The merchant then funds their side of the smart contract, if applicable. Once the contract is fully funded, the funds are locked in. At this point, the merchant runs `zkAbacus.\Activate()` to generate the initial payment tag and sends the customer the message `open_m`, which contains the payment tag.

When the customer receives and verifies the payment tag, the channel is open and they are ready to make payments with the [payments protocol](3-channel-payments.md).

        +-------+                           +-------+
        |       |---------  prep_c  ------->|       |
        |       |<--------  prep_m  --------|       |
        |       |                           |       |
        |       |---------  init_c  ------->|       |
        | Cust  |<--------  init_m  --------| Merch |
        |       |                           |       |
        |       |---------- open_c -------->|       |
        |       |<--------- open_m ---------|       |
        |       |                           |       |
        +-------+                           +-------+

        

## Global defaults
* [`int`:`selfDelay`] 
* [`int`:`minimum_depth`]

`selfDelay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 
`minimum_depth` sets the minimum number of confirmations for the funding to be considered final.

## The `prep_c` Message
1. type: (`prep_c`)
2. data: 
    * [`string`:`cid_c`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]
    * [`address`:`cust_addr`]
    * [`key`:`cust_pk`]
    * [`bytes`:`merch_pk_hash`]

Here, `merch_pk_hash` is the SHA3-256 hash of the merchant's public parameters including their Pointcheval-Sanders public key and Tezos account information. This ensures that the customer has the correct merchant details before attempting to open a channel on chain. 
XX TODO: define this hash.

### Requirements
The customer:
   - Generates `cid_c` randomly using a secure RNG. 

Upon receipt, the merchant:
  - Checks that `cid_c` is a valid string and `bal_cust_0` ≥ 0 and `bal_merch_0` ≥ 0 are positive integers.
  - Checks that `merch_pk_hash` is correct. 

## The `prep_m` Message
1. type: (`prep_m`)
2. data:
    * [`string`:`cid_m`]

### Requirements
Before sending, the merchant:
  - Ensures `cid_m` is generated randomly and is unique for each channel.
  - Sets `cid` to H(`cid_c`, `cid_m`, `cust_pk`, `merch_pk`, `merch_PS_pk`) where `cust_pk`, `merch_pk` refer to the customer and merchant's Tezos account public keys respectively, and `merch_PS_pk` refers to the merchant's public PS public keys.

Upon receipt, the customer:
  - Checks that `cid_m` is in the expected range. If yes, set `cid` to H(`cid_c`, `cid_m`, `cust_pk`, `merch_pk`, `merch_PS_pk`).

## The `init_c` Message
The customer initiates `zkAbacus.Initialize()` by sending the merchant `init_c`.

1. type: (`init_c`)
2. data: 
    * [`bls12_381_g2`:`A'`]
    * [`bls12_381_g2`:`A''`]
    * [`bls12_381_g2`/`bls12_381_g1`/`bls12_381_fr`:`Pi`]

### Requirements

## The `init_m` Message
1. type: (`init_m`)
2. data: 
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

Here, `closing_signature` is the merchant's closing authorization signature over the initial state.

Upon receipt, the customer:
  - Verifies the merchant's `closing_signature` on the initial state.
## The `open_c` Message
This message tells the merchant that the channel has been originated and the customer's side of the channel has been funded. For more information about this process, please refer to [2-contract-origination.md](2-contract-origination.md).

1. type: (`open_c`)
2. data: 
    * [`address`:`contract-id`]

The `cid` lets the merchant know which channel the customer is referring to, and `contract-id` allows the merchant to search for the contract on-chain and verify its contents.

### Requirements

The customer:
  - Ensures that the funds have been confirmed on chain for at least `minimum_depth`.

Upon receipt, the merchant:
  - Checks that the originated contract `contract-id` has the exact same code as the zkchannels contract.
  - Checks that the on-chain storage of `contract-id` is exactly as expected for channel `cid` (including that the customer's side has been funded).
  - Checks that the contract storage `status` has been set to `OPEN` (denoted as `1`) for at least `minimum_depth` blocks.

For a dual funded channel, the merchant will fund their side of the channel (see [2-contract-origination.md](2-contract-origination.md)).
  ## The `open_m` Message

1. type: (`open_m`)
2. data: 
    * [`json`:`payment_tag`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

### Requirements
When the contract is fully funded, the `status` will change to `OPEN` (denoted as `1`) indicating that the funds are locked in. At this point the merchant waits until this status has been stable for `minimum_depth` blocks before sending `funding_locked` to the customer.

Upon receipt, the customer:
  - verifies the `payment_tag`.