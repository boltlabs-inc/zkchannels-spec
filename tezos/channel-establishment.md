# Channel Establishment

  * [Overview](#Overview)
    * [The `open_c` Message](#the-`open_c`-Message)
    * [The `open_m` Message](#the-`open_m`-Message)
    * [The `init_c` Message](#the-`init_c`-Message)
    * [The `init_m` Message](#the-`init_m`-Message)
    * [The `funding_confirmed` Message](#the-`funding_confirmed`-Message)
    * [The `funding_ack` Message](#the-`funding_ack`-Message)
    * [The `activate_c` Message](#the-`activate_c`-Message)
    * [The `activate_m` Message](#the-`activate_m`-Message)

## Overview
TODO zkchannels-spec#5: Add high level overview of establish

        +-------+                              +-------+
        |       |--(1)-------  open_c  ------->|       |
        |       |<-(2)-------  open_m  --------|       |
        |       |                              |       |
        |   A   |--(3)-------- init_c -------->|   B   |
        |       |<-(4)-------- init_m ---------|       |
        |       |                              |       |
        |       |--(5)-- funding_confirmed --->|       |
        |       |<-(6)---- funding_ack --------|       |
        |       |                              |       |
        |       |--(5)------ activate_c ------>|       |
        |       |<-(6)------ activate_m -------|       |
        +-------+                              +-------+

        - where node A is the 'customer' and node B is the 'merchant'

## Global defaults
* [`int`:`selfDelay`] 
* [`int`:`minimum_depth`]
* [`bls12_381_g2`:`g2`]

`selfDelay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 

`minimum_depth` sets the minimum number of confirmations for the funding to be considered final.

### The `open_c` Message

1. type: (`open_c`)
2. data: 
    * [`string`:`cid_p`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]
#### Requirements

The customer:
  - Ensures `cid_p` is generated randomly and is unique for each channel.

The merchant:
  - Checks that `cid_p` , `bal_cust_0` ≥ 0, and `bal_merch_0` ≥ 0 are in the expected domain.

### The `open_m` Message

1. type: (`open_m`)
2. data:
    * [`bool`:`accept`]
    * [`string`:`cid_m`]

#### Requirements

The merchant:
  - Ensures `cid_m` is generated randomly and is unique for each channel.

Both the customer and merchant:
  - set `cid` to H(`cid_p`, `cid_m`)

### The `init_c` Message

1. type: (`init_c`)
2. data: 
    * [`string`:`custAddr`]
    * [`string`:`custPk`]

#### Requirements

### The `init_m` Message

1. type: (`init_m`)
2. data: 
    * [`address`:`merchAddr`]
    * [`key`:`merchPk`]
    * [`bls12_381_g2`:`merchPk0`]
    * [`bls12_381_g2`:`merchPk1`]
    * [`bls12_381_g2`:`merchPk2`]
    * [`bls12_381_g2`:`merchPk3`]
    * [`bls12_381_g2`:`merchPk4`]
    * [`bls12_381_g2`:`merchPk5`]
    * [`bls12_381_fr`:`hashCloseB`]
    * [`json`:`closing_signature`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

`hashCloseB` is the set to H(`merch_str`), where `merch_str` is a unique string set by the merchant.
#### Requirements

### The `funding_confirmed` Message

1. type: (`funding_confirmed`)
2. data: 
    * [`string`:`cid`]
    * [`string`:`contract-id`]

#### Requirements

The customer:
  - Ensures that the funds have been confirmed on chain for at least `minimum_depth` + 1 blocks.

### The `funding_ack` Message

1. type: (`funding_ack`)
2. data: 
    * [`string`:`accept`] (XX: check what this msg should be)

#### Requirements

The merchant:
  - Checks that the originated contract `contract-id` has the exact same code as the zkchannels contract.
  - Checks that the on-chain storage of `contract-id` is exactly as expected for channel `cid` (including that the customer's side has been funded).
  - Checks that the contract storage `status` has not changed for at least `minimum_depth` blocks before sending this message.
  - If it is a dual funded channel, the merchant must fund their side of the channel.

### The `activate_c` Message

1. type: (`activate_c`)
2. data: 
    * [`bls12_381_fr`:`cid`]
    * [`string`:`cid_p`]

#### Requirements

The customer:
  - Waits until the contract storage field `status` has been set to `1` (meaning the funding has been locked in) for at least `minimum_depth` blocks before sending this message.

### The `activate_m` Message

1. type: (`activate_m`)
2. data: 
    * [`json`:`payment_tag`]
      * [`bls12_381_g1`:`s1`]
      * [`bls12_381_g1`:`s2`]

#### Requirements

