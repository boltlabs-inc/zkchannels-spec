# Channel Establishment
  * [Overview](#Overview)
  * [Global defaults](#global-defaults)
  * [Message Specifications](#message-specifications)
    * [The `open_c` Message](#the-open_c-message)
    * [The `open_m` Message](#the-open_m-message)
    * [The `init_c` Message](#the-init_c-message)
    * [The `init_m` Message](#the-init_m-message)
    * [The `funding_confirmed` Message](#the-funding_confirmed-message)
    * [The `activate` Message](#the-activate-message)
  * [Contract Origination and Funding](2-contract-origination.md)
## Prerequisites
The merchant has completed the [setup](#1-setup.md) phase, and the customer and merchant have established a communication session.

The customer has [obtained the merchant’s setup information](1-setup.md) out of band beforehand including the [merchant's public parameters](1-setup.md#Public-parameters). The customer must verify the merchant's public parameters are well-formed:
* The merchant blind signing public key `merch_PS_pk` must consist of a valid Pointcheval Sanders public key of the expected length.
* The range proof parameters `range_proof_params` must consist of a valid Pointcheval Sanders key of the expected length and signatures on the appropriate integer range.
* The merchant EdDSA public key `merch_pk` must be a valid EdDSA key for the curve specified by `tezos-client` and the merchant address `merch_addr` must be a Tezos tz1 address correctly derived from `merch_pk`. 

## Overview
Channel establishment is a three round protocol.

In the first round, the customer sends the `open_c` message, which contains information about the initial state of the proposed channel. If the merchant agrees to open the proposed channel, they reply with the message `open_m`, which contains the merchan'ts contribution to the channel identifier. At the end of this round, the customer and merchant have exchanged enough information to compute the channel identifer `cid`, which acts as the unique channel identifier for the on-chain Tezos escrow account and off-chain `zkAbacus` channel.

In the next round, the customer and merchant initialize the `zkAbacus` channel by running `zkAbacus.Initialize()` on the previously established public parameters. In this subroutine, the customer sends `init_c`, which consists of a (hiding) commitment to the intial state and a zero-knowledge proof of correctness. In return, the merchant sends `init_m`, which contains an initial closing authorization signature for the customer. 

For the final round, the customer originates the contract and funds their side of the channel (see [2-contract-origination.md](2-contract-origination.md) for more information). The customer sends `funding_confirmed` to the merchant, which contains the contract id `contract-id`. The merchant checks the corresponding contract and initial storage for the expected values. The merchant then funds their side of the smart contract, if applicable. Once the contract is fully funded, the funds are locked in. At this point, the merchant runs `zkAbacus.Activate()` to generate the initial payment tag and sends the customer the message `activate`, which contains the payment tag.

Upon completion of `zkAbacus.Activate()`, the channel is open and ready for [payments](3-channel-payments.md).  Details for `zkAbacus.Initialize()` maybe found here. (XX reference)



        +-------+                           +-------+
        |       |---------  open_c  ------->|       |
        |       |<--------  open_m  --------|       |
        |       |                           |       |
        |       |---------  init_c  ------->|       |
        | Cust  |<--------  init_m  --------| Merch |
        |       |                           |       |
        |       |---- funding_confirmed --->|       |
        |       |<------- activate ---------|       |
        |       |                           |       |
        +-------+                           +-------+

        

## Global defaults
* [`int`:`self_delay`] 
* [`int`:`minimum_depth`]

`self_delay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 
`minimum_depth` sets the minimum number of confirmations for the funding to be considered final.

## Message Specifications

### The `open_c` Message
1. type: (`open_c`)
2. data: 
    * [`string`:`cid_c`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]
    * [`address`:`cust_addr`]
    * [`key`:`cust_pk`]
    * [`string`: `merch_pp_hash`]

#### Requirements
The customer:
   - Generates `cid_c` randomly using a secure RNG. 

Upon receipt, the merchant:
  - Checks that `cid_c` is a valid string and `bal_cust_0` ≥ 0 and `bal_merch_0` ≥ 0 are positive integers.
  - Checks that `pk_cust` is a valid EdDSA public key for the curve specified by `tezos-client` and that `cust_addr` is a valid Tezos tz1 address that is correctly derived from `pk_cust`.
  - Checks that `merch_pp_hash` is correct with respect to `SHA3-256(merch_PS_pk, merch_addr, merch_pk)` and rejects channel open request if not.

### The `open_m` Message
1. type: (`open_m`)
2. data:
    * [`string`:`cid_m`]

#### Requirements
Before sending, the merchant:
  - Generates `cid_m` randomly using a secure RNG.
  - Sets `cid` to `SHA3-256(cid_c, cid_m, cust_pk, merch_pk, merch_PS_pk)` where `cust_pk`, `merch_pk` refer to the customer and merchant's Tezos account public keys respectively, and `merch_PS_pk` refers to the merchant's public PS public keys.

Upon receipt, the customer:
  - Checks that `cid_m` is a valid string. If yes, sets `cid` to `SHA3-256(cid_c, cid_m, cust_pk, merch_pk, merch_PS_pk)`.

### The `init_c` Message
The customer runs the `zkAbacus.Initialize()` with the following inputs: `cust_pk`, `cid`, `bal_cust` and `bal_merch`. The customer sends an `init_c` message which consists of the output of `Initialize()`: a pair of (hiding) commitments to the intial state and a zero-knowledge proof.

1. type: (`init_c`)
2. data:
    * [`string`:`cid`]
    * [`bls12_381_g1`:`close_state_commitment`]
    * [`bls12_381_g1`:`state_commitment`]
    * [`(bls12_381_g1, bls12_381_g1, Vec<bls12_381_fr>): pay_proof`]

#### Requirements
Upon receipt, the merchant checks the correctness of the `cid` in the `init_c` message and continues as specified in `zkAbacus.Initialize()`. 

### The `init_m` Message
The merchant sends back an `init_m` message that consists of a closing authorization signature `closing_signature` on the initial state if `zkAbacus.Initialize()` was successful.

1. type: (`init_m`)
2. data: 
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Requirements
Upon receipt, the customer:
  - Verifies `closing_signature` and continues as specified in `zkAbacus.Initialize()`.

### The `funding_confirmed` Message
This message tells the merchant that the contract has been originated and the customer's side of the escrow account has been funded. For more information about this process, please refer to [2-contract-origination.md](2-contract-origination.md).

1. type: (`funding_confirmed`)
2. data: 
    * [`address`:`contract-id`]


#### Requirements

The customer:
  - Ensures that the funds have been confirmed on chain for at least `minimum_depth`.

Upon receipt, the merchant:
  - Checks that the originated contract `contract-id` contains the expected zkchannels [contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz).
  - Checks that the on-chain storage of `contract-id` is exactly as expected for channel `cid` (including that the customer's side has been funded). Specifically, checks that:
    - The values stored in the following fields match the merchant's PS public key values:`g2`, `X`, `Y1`, `Y2`, `Y3`, `Y4`, `Y5`.
    - The merchant's tezos address and public key match the fields `merch_addr` and  `merch_pk`, respectively.
    - The `self_delay` field in the contract matches the global default. 
    - The `close` field matches the merchant's `close` flag.
    - `custFunding` and `merchFunding` match the initial balances `bal_cust_0` and `bal_merch_0`, respectively.
  - In the customer-funded case, checks that the contract storage `status` has been set to `OPEN` (denoted as `1`) for at least `minimum_depth` blocks.
  - In the dual-funded case, the merchant funds their side of the escrow account (see [2-contract-origination.md](2-contract-origination.md)).

  ### The `activate` Message
  This `activate` message consists of the sole message of `zkAbacus.Activate()`.

1. type: (`activate`)
2. data: 
    * [`json`:`payment_tag`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Requirements
Before sending, the merchant:
  - Waits until the contract storage status has been set to `OPEN` (denoted as `1`) for `minimum_depth` blocks.

If the customer fails to fund their side of the contract within 180 minutes of sending `funding_confirmed`, the merchant will abort. If the decision to abort occurs after the merchant has already funded their side of the channel, call the `@reclaimFunding` entrypoint in order to retrieve the merchant's initial funding.

Upon receipt, the customer:
  - In the dual-funded case, verifies that contract storage status has been `OPEN` for `minimum_depth` blocks. If this check fails, the customer should reclaim their funding by initiating a unilateral close.
  - Verifies `payment_tag` which is the output of `zkAbacus.Activate()`. If this check fails, the customer should reclaim their funding by initiating a unilateral close.
