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
## Prerequisites
The merchant has completed the [setup](#1-setup.md) phase, and the customer and merchant have established a communication session.

The customer has [obtained the merchant’s setup information](1-setup.md) out of band beforehand including the [merchant's public parameters](1-setup.md#Public-parameters). The customer must verify the merchant's public parameters are well-formed:
* The merchant blind signing public key `merch_PS_pk` must consist of a valid Pointcheval Sanders public key of the expected length.
* The range proof parameters `range_proof_params` must consist of a valid Pointcheval Sanders key of the expected length and signatures on the appropriate integer range.
* The pedersen commitment parameters must be well-formed from `G1` and of th expected length.
* The merchant EdDSA public key `merch_pk` must be a valid EdDSA key for the curve specified by `tezos-client` and the merchant address `merch_addr` must be a Tezos tz1 address correctly derived from `merch_pk`. 

## Overview
Channel establishment is a three round protocol.

In the first round, the customer sends the `open_c` message, which contains information about the initial state of the proposed channel. If the merchant agrees to open the proposed channel, they reply with the message `open_m`, which contains the merchan'ts contribution to the channel identifier. At the end of this round, the customer and merchant have exchanged enough information to compute the channel identifer `cid`, which acts as the unique channel identifier for the on-chain Tezos escrow account and off-chain `zkAbacus` channel.

In the next round, the customer and merchant initialize the `zkAbacus` channel by running `zkAbacus.Initialize()` on the previously established public parameters. In this subroutine, the customer sends `init_c`, which consists of a (hiding) commitment to the intial state and a zero-knowledge proof of correctness. In return, the merchant sends `init_m`, which contains an initial closing authorization signature for the customer. 

For the final round, the customer originates the contract and funds their side of the channel (see [5-tezos-escrowagent.md](5-tezos-escrowagent.md#contract-origination-and-funding) for more information). The customer sends `funding_confirmed` to the merchant, which contains the contract id `contract-id`. The merchant checks the corresponding contract and initial storage for the expected values. The merchant then funds their side of the smart contract, if applicable. Once the contract is fully funded, the funds are locked in. At this point, the merchant runs `zkAbacus.Activate()` to generate the initial payment tag and sends the customer the message `activate`, which contains the payment tag.

Upon completion of `zkAbacus.Activate()`, the channel is open and ready for [payments](3-channel-payments.md).  Details for `zkAbacus.Initialize()` may be found in section 3.3.2 of the [zkChannels Protocol](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf).




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
* [`int`:`required_confirmations`]

`self_delay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 
`required_confirmations` sets the minimum number of confirmations for the funding to be considered final.

## Message Specifications

### The `open_c` Message
Channel establishment begins with the customer sending an `open_c` message to the merchant which describes properties about the channel it wishes to open. The proposed initial balances for the customer and merchant are denoted by `bal_cust_0` and `bal_merch_0`, respectively. `cid_c` is the customer's randomly generated contribution to what will form the channel id (`cid`). `cust_addr` is the customer's tezos account address, and `cust_pk` is the customer's tezos account public key. 

1. type: (`open_c`)
2. data: 
    * [`string`:`cid_c`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]
    * [`address`:`cust_addr`]
    * [`key`:`cust_pk`]
    * [`string`: `merch_pp_hash`]

#### Requirements
The customer generates `cid_c` randomly using a secure RNG. 

#### Customer abort conditions
The customer checks that `merch_addr` is an implicit Tezos account (tz1 address), and not a smart contract address (KT1 address). If `merch_addr` is not an implicit Tezos account, abort. This is to ensure that dispersed payouts on channel closure cannot be failed by a smart contract refusing payments.

#### Merchant abort conditions
Upon receipt, the merchant checks that the following are true. If any are false, the merchant aborts:
  - `cid_c` is a valid string and `bal_cust_0` ≥ 0 and `bal_merch_0` ≥ 0 are positive integers.
  - `cust_pk` is a valid EdDSA public key for the curve specified by `tezos-client` and that `cust_addr` is a valid Tezos tz1 address that is correctly derived from `cust_pk`.
  - `merch_pp_hash` is correct with respect to `SHA3-256(merch_PS_pk, merch_addr, merch_pk)` and rejects channel open request if not.
  - `cust_addr` is an implicit Tezos account (tz1 address), and not a smart contract address (KT1 address). This is to ensure that dispersed payouts on channel closure cannot be failed by a smart contract refusing payments.

### The `open_m` Message
1. type: (`open_m`)
2. data:
    * [`string`:`cid_m`]

#### Requirements
Before sending, the merchant:
  - Generates `cid_m` randomly using a secure RNG.
  - Sets `cid` to `SHA3-256(cid_c, cid_m, cust_pk, merch_pk, merch_PS_pk)` where `cust_pk`, `merch_pk` refer to the customer and merchant's Tezos account public keys respectively, and `merch_PS_pk` refers to the merchant's public PS public keys.

#### Customer abort conditions
Upon receipt, the customer checks the that `cid_m` is a valid string. If it is, the customer sets `cid` to `SHA3-256(cid_c, cid_m, cust_pk, merch_pk, merch_PS_pk)`. If it is not, the customer aborts.

### The `init_c` Message
The customer runs the `zkAbacus.Initialize()` with the following inputs: `cust_pk`, `cid`, `bal_cust` and `bal_merch`. The customer sends an `init_c` message which consists of the output of `Initialize()`: a pair of (hiding) commitments to the intial state and a zero-knowledge proof.

1. type: (`init_c`)
2. data:
    * [`string`:`cid`]
    * [`bls12_381_g1`:`close_state_commitment`]
    * [`bls12_381_g1`:`state_commitment`]
    * [`(bls12_381_g1, bls12_381_g1, Vec<bls12_381_fr>): establish_proof`]

#### Merchant abort conditions
Upon receipt, the merchant checks the correctness of the `cid` in the `init_c` message and continues as specified in `zkAbacus.Initialize()`. If `cid` is incorrect, the merchant aborts.

### The `init_m` Message
The merchant sends back an `init_m` message that consists of a closing authorization signature `closing_signature` on the initial state if `zkAbacus.Initialize()` was successful.

1. type: (`init_m`)
2. data: 
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Customer abort conditions
Upon receipt, the customer verifies `closing_signature`. If the signature is valid, the customer continues as specified in `zkAbacus.Initialize()`. If the siganture is invalid, the customer aborts.

### The `funding_confirmed` Message
This message tells the merchant that the contract has been originated and the customer's side of the escrow account has been funded. The customer sends the contract identifier, `contract-id`, as well as the origination block height, `block-height`, to make it easier for the merchant to find the origination operation on chain. For more information about this process, please refer to [5-tezos-escrowagent.md](5-tezos-escrowagent.md#contract-origination-and-funding) for more information).

1. type: (`funding_confirmed`)
2. data: 
    * [`address`:`contract-id`]
    * [`int`:`originated-block-height`]

#### Requirements

The customer:
  - Ensures that the funds have been confirmed on chain for at least `required_confirmations` before sending `funding_confirmed`.

#### Merchant abort conditions
Upon receipt, the merchant checks that the following are true. If any are false, the merchant aborts:
  - The originated contract `contract-id` contains the expected zkchannels [contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz).
  - The on-chain storage of `contract-id` at `originated-block-height` is exactly as expected for channel `cid` (including that the customer's side has been funded). Specifically, checks that:
    - The contract storage contains the merchant's Pointcheval Sanders public key.
    - The customer's tezos address and public key match the fields `cust_addr` and  `cust_pk`, respectively.
    - The merchant's tezos address and public key match the fields `merch_addr` and  `merch_pk`, respectively.
    - The `self_delay` field in the contract matches the value specified in the [global defaults](1-setup.md#Global-defaults). 
    - The `close` field in the contract matches the merchant's `close` flag defined as defined in the [global defaults](1-setup.md#Global-defaults). The `close` flag represents a fixed scalar used by the merchant to differentiate closing state and state.
    - `custFunding` and `merchFunding` match the initial balances `bal_cust_0` and `bal_merch_0`, respectively.
    - The `status` field of the contract is set to `0`, which corresponds to `AWAITING_FUNDING`.
    - The `context-string` is set to "zkChannels mutual close", as defined in the [global defaults](1-setup.md#Global-defaults). 

Before proceeding, the merchant waits until the customer's side of the contract has been funded for at least `required_confirmations` blocks. If this has not occured within an arbitrary timeout period (longer than the time expected to satisfy `required_confirmations`), the merchant stops waiting for the customer to fund the channel and aborts.

In the dual-funded case, the merchant funds their side of the escrow account (see [5-tezos-escrowagent.md](5-tezos-escrowagent.md#contract-origination-and-funding) for more information).

  ### The `activate` Message
  This `activate` message consists of the sole message of `zkAbacus.Activate()`.

1. type: (`activate`)
2. data: 
    * [`json`:`payment_tag`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

Before sending, the merchant:
  - Waits until the contract storage status has been set to `OPEN` (denoted as `1`) for `required_confirmations` blocks.

#### Customer abort conditions
Upon receipt, the customer checks that the following are true. If any are false, the customer aborts:
  - In the dual-funded case, the customer must wait for the contract storage status has been `OPEN` for `required_confirmations` blocks before proceeding. If this has not occured within an arbitrary timeout period (longer than the time expected to satisfy `required_confirmations`), the customer stops waiting for the merchant to fund the channel and aborts. To reclaim the funds that the customer had added, the customer calls the `reclaimFunding` entrypoint on the contract.
  - The `payment_tag`, which is the output of `zkAbacus.Activate()`, is valid. If not, the customer aborts by initiating a unilateral close.
