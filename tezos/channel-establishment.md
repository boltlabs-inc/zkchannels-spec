# Channel Establishment

  * [Overview](#Overview)
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
  
## Overview
(XX: TODO - add overall summary about messages sent during Establish. Something similar to 'Channel Establishment' section in LN spec)

        +-------+                              +-------+
        |       |--(1)-------  open_c  ------->|       |
        |       |<-(2)-------  open_c  --------|       |
        |       |                              |       |
        |   A   |--(3)-------- init_c -------->|   B   |
        |       |<-(4)-------- init_m ---------|       |
        |       |                              |       |
        |       |--(5)-- funding_confirmed --->|       |
        |       |<-(6)--- funding_locked ------|       |
        +-------+                              +-------+

        - where node A is the 'customer' and node B is the 'merchant'

## Global defaults
* [`int`:`selfDelay`] 
* [`int`:`minimum_depth`]

`selfDelay` sets the length of the dispute period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. 

`minimum_depth` sets the minimum number of confirmations for the funding to be considered final.

### The `open_c` Message

1. type: XX (`open_c`)
2. data: (XX: Check)
    * [`string`:`cid_p`]
    * [`int`:`bal_cust_0`]
    * [`int`:`bal_merch_0`]

#### Requirements

The customer MUST:
  - Ensure `cid_p` is generated randomly and is unique for each channel.

The merchant MUST:
  - Check that `cid_p` , `bal_cust_0` ≥ 0, and `bal_merch_0` ≥ 0 are in the expected domain.

### The `open_c` Message

1. type: XX (`open_m`)
2. data: (XX: Check)
    * [`bool`:`accept`]
    * [`string`:`cid_m`]

#### Requirements

The merchant MUST:
  - Ensure `cid_p` is generated randomly and is unique for each channel.

Both the customer and merchant MUST:
  - set `cid` to H(`cid_p`, `cid_m`)

### The `init` Message

1. type: XX (`init`)
2. data: (XX: Fill in)

#### Requirements

### The `funding_confirmed` Message

1. type: XX (`funding_confirmed`)
2. data: 
    * [`string`:`cid`]
    * [`string`:`contract-id`]

#### Requirements

The customer MUST:
  - Ensure that the funds have been confirmed on chain for at least `minimum_depth` blocks.
(XX: May decide that this gets sent only after origination has been confirmed)

The merchant MUST:
  - Check that the originated contract `contract-id` has the exact same code as the zkchannels contract.
  - Check that the on-chain storage of `contract-id` is exactly as expected for channel `cid`.

### The `funding_locked` Message

1. type: XX (`funding_locked`)
2. data: 
    * [`string`:`cid`]

#### Requirements

The merchant MUST:
  - Ensure that the contract storage field `status` has been set to `1` (meaning the funding has been locked in) for at least `minimum_depth` blocks.

The customer SHOULD:
  - Wait until the contract storage field `status` has been set to `1` (meaning the funding has been locked in) for at least `minimum_depth` blocks.

## Activate

## Unlink

# Pay

# Close