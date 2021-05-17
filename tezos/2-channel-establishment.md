# Channel Establishment

  * [Overview](#Overview)
  * [Global defaults](#global-defaults)
  * [The `prep_c` Message](#the-`init_c`-Message)
  * [The `prep_m` Message](#the-`init_m`-Message)
  * [The `init_c` Message](#the-`init_c`-Message)
  * [The `init_m` Message](#the-`init_m`-Message)
  * [The `open_c` Message](#the-`open_c`-Message)
  * [The `open_m` Message](#the-`open_m`-Message)

## Overview
After the merchant has completed the [setup](#1-setup.md) phase, and the customer and merchant have established a communication session, channel establishment may begin. 

Channel establishment begins with the customer sending the `init_c` message, containing information about the proposed initial state of the channel contract. If the merchant agrees to the proposed channel, they reply with `init_m`, containing the initial closing authorization signature (`closing_signature`).  

At this point, the customer and merchant have exchanged enough information to compute a channel is, `cid`, which will act as the unique channel identifier on and off chain. The customer originates the contract with the initial state and funds their side of the channel. The customer sends `open_c` to the merchant, containing the contract id. The merchant checks that the contract on chain matches up with what they were expecting (the contract and inital storage). The merchant then funds their side of the smart contract. Once the contract is fully funded, the funds are locked in and the merchant sends the customer `open_m` with the first payment tag.

When the customer receives and verifies the payment tag, the channel is open and they are ready to make payments with the [payments protocol](3-channel-payments.md).

        +-------+                              +-------+
        |       |--(1)-------  prep_c  ------->|       |
        |       |<-(2)-------  prep_m  --------|       |
        |       |                              |       |
        |       |--(1)-------  init_c  ------->|       |
        |       |<-(2)-------  init_m  --------|       |
        |   A   |                              |   B   |
        |       |--(3)-------- open_c -------->|       |
        |       |<-(4)-------- open_m ---------|       |
        |       |                              |       |
        +-------+                              +-------+

        - where node A is the 'customer' and node B is the 'merchant'

## Global defaults
* [`int`:`selfDelay`] 
* [`int`:`minimum_depth`]

`selfDelay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 
`minimum_depth` sets the minimum number of confirmations for the funding to be considered final.

### The `prep_c` Message
1. type: (`prep_c`)
2. data: 
    * [`string`:`cid_c`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]
    * [`address`:`cust_addr`]
    * [`key`:`cust_pk`]
    * [`bytes`:`merch_pk_hash`]

Here, `merch_pk_hash` is the hash of the merchant's public parameters including their PS key and Tezos account information. This ensures that the customer has the correct merchant details before attempting to open a channel on chain. 

#### Requirements
The customer:
  - Needs to have obtained the merchant’s setup information (signature parameters and tezos account details) out of band beforehand.
  - Ensures `cid_c` is generated randomly and is unique for each channel.

Upon receipt, the merchant:
  - Checks that `cid_c` , `bal_cust_0` ≥ 0, and `bal_merch_0` ≥ 0 are in the expected domain.
  - Checks that `merch_pk_hash` is correct.

### The `prep_m` Message
1. type: (`prep_m`)
2. data:
    * [`string`:`cid_m`]

#### Requirements
Both the customer and merchant:
  - Set `cid` to H(`cid_c`, `cid_m`, `cust_pk`, `merch_pk`, `merch_PS_pk`) where `cust_pk`, `merch_pk` refer to the customer and merchant's Tezos account public keys respectively, and `merch_PS_pk` refers to the merchant's public PS public keys.

Before sending, the merchant:
  - Ensures `cid_m` is generated randomly and is unique for each channel.

Upon receipt, the customer:
  - Verifies the merchant's `closing_signature` on the initial state.
  

### The `init_c` Message
1. type: (`init_c`)
2. data: 

### The `init_m` Message
1. type: (`init_m`)
2. data: 
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

Here, `closing_signature` is the merchant's closing authorization signature over the initial state.

### The `open_c` Message
This message tells the merchant that the channel has been originated and the customer's side of the channel has been funded. For more information about this process, please refer to [2-contract-origination.md](2-contract-origination.md).

1. type: (`open_c`)
2. data: 
    * [`address`:`contract-id`]

The `cid` lets the merchant know which channel the customer is referring to, and `contract-id` allows the merchant to search for the contract on-chain and verify its contents.

#### Requirements

The customer:
  - Ensures that the funds have been confirmed on chain for at least `minimum_depth`.

Upon receipt, the merchant:
  - Checks that the originated contract `contract-id` has the exact same code as the zkchannels contract.
  - Checks that the on-chain storage of `contract-id` is exactly as expected for channel `cid` (including that the customer's side has been funded).
  - Checks that the contract storage `status` has been set to `OPEN` (denoted as `1`) for at least `minimum_depth` blocks.

For a dual funded channel, the merchant will fund their side of the channel (see [2-contract-origination.md](2-contract-origination.md)).
  ### The `open_m` Message

1. type: (`open_m`)
2. data: 
    * [`json`:`payment_tag`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Requirements
When the contract is fully funded, the `status` will change to `OPEN` (denoted as `1`) indicating that the funds are locked in. At this point the merchant waits until this status has been stable for `minimum_depth` blocks before sending `funding_locked` to the customer.

Upon receipt, the customer:
  - verifies the `payment_tag`.