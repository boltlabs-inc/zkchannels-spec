# Channel Establishment
  * [Overview](#overview)
  * [Global defaults](#global-defaults)
  * [Message Specifications](#message-specifications)
    * [The `open_c` Message](#the-open_c-message)
    * [The `open_m` Message](#the-open_m-message)
    * [The `init_c` Message](#the-init_c-message)
    * [The `init_m` Message](#the-init_m-message)
    * [The `funding_confirmed` Message](#the-funding_confirmed-message)
    * [The `activate` Message](#the-activate-message)
## Prerequisites
The merchant has completed the [setup](1-setup.md#merchant-setup) phase, and the customer and merchant have established a communication session.

The customer has [obtained the merchant’s setup information](1-setup.md#publishing-public-parameters) out of band. The customer must verify the merchant's public parameters are well-formed and valid:
<<<<<<< HEAD
* The merchant blind signing public key `merchant_zkabacus_public_key` must consist of a valid Pointcheval Sanders public key of the expected length with components in the BLS12-381 pairing subgroups G1 and G2.
=======
* The merchant blind signing public key `merchant_blind_public_key` must consist of a valid Pointcheval Sanders public key of the expected length with components in the BLS12-381 pairing subgroups G1 and G2.
>>>>>>> a8faf50d916f8ab6c5460f8d681ce366008aa0e4
* The range proof parameters `range_proof_params` must consist of a valid Pointcheval Sanders key of the expected length with components in the BLS12-381 pairing subgroup G1, and valid signatures on the appropriate integer range.
* The revocation lock commitment parameters `revlock_com_params` must be well-formed Pedersen parameters of the expected length, and consist of elements in the BLS12-381 pairing subgroup G1.
* The merchant EdDSA public key `merch_pk` must be a valid EdDSA key for the curve specified by `tezos-client` and the merchant address `merchant_address` must be a Tezos tz1 address correctly derived from `merch_pk`. 

The customer should ensure they have a Tezos implicit account with balance sufficient to both contribute the desired amount to the zkChannel and pay the [operations fees](5-tezos-escrowagent.md#operation-fees) needed to originate, fund, and call the appropriate entry points of the corresponding smart contract. We recommend 2 tez based on our [contract benchmarks](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results) on testnet.

## Overview
Channel establishment is a three round protocol.

In the first round, the customer sends the `open_c` message, which contains information about the initial state of the proposed channel. If the merchant agrees to open the proposed channel, they reply with the message `open_m`, which contains the merchan'ts contribution to the channel identifier. At the end of this round, the customer and merchant have exchanged enough information to compute the channel identifer `channel_id`, which acts as the unique channel identifier for the on-chain Tezos escrow account and off-chain `zkAbacus` channel. 

In the next round, the customer and merchant initialize the `zkAbacus` channel by running `zkAbacus.Initialize()` on the previously established public parameters. In this subroutine, the customer sends `init_c`, which consists of a (hiding) commitment to the intial state and a zero-knowledge proof of correctness. In return, the merchant sends `init_m`, which contains an initial closing authorization signature for the customer. 

For the final round, the customer originates the contract and funds their side of the channel as specified in [Contract Origination and Funding](5-tezos-escrowagent.md#contract-origination-and-funding). The customer sends `funding_confirmed` to the merchant, which contains the contract identifier `contract-id`. The merchant checks the corresponding contract and initial storage for the expected values. The merchant then funds their side of the smart contract, if applicable. Once the contract is fully funded, the funds are locked in. At this point, the merchant runs `zkAbacus.Activate()` to generate the initial payment tag and sends the customer the message `activate`, which contains the payment tag. Upon completion of `zkAbacus.Activate()`, the channel is open and ready for [payments](3-channel-payments.md). 

 Details for `zkAbacus` may be found in Chapter 3.3.3 of the [zkChannels Protocol document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review-v1/zkchannels-protocol-spec-v3.1.pdf).




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

        

## Global Defaults
* [`int`:`self_delay`]: The default timeout length applied to the customer closing entrypoint `custClose` and the merchant expiry entrypoint `expiry`. The value is interpreted in sections.
* [`int`:`required_confirmations`]: The minimum number of confirmations for an operation to be considered final.

## Message Specifications

### The `open_c` Message
The `open_c` message is sent from the customer to the merchant and is formed as follows:

1. type: `open_c`
2. data: 
    * [`string`:`channel_id_c`]: Customer randomness contribution to the channel identifer.
    * [`int`:`customer_balance`]: The proposed initial customer balance.
    * [`int`:`merchant_balance`]: The proposed initial merchant balance.
    * [`address`:`customer_address`]: The customer's Tezos tz1 account address.
    * [`key`:`cust_pk`]: The customer's Tezos EdDSA public key.
    * [`string`: `merch_pp_hash`]: A hash of merchant public parameters as specified in [Merchant Setup](1-setup.md#merchant-setup).

#### Customer Requirements

The customer, before sending:
- Retrieves the merchant public parameters and checks these parameters are well-formed and valid as specified [above](#prerequisites).
- Generates `channel_id_c` randomly using a secure RNG. 


#### Merchant Requirements

Upon receipt, the merchant checks that the following are true. If any are false, the merchant aborts:
  - Checks `channel_id_c` is valid customer randomness for the contract identifier.
  - Checks `customer_balance` ≥ 0 and `merchant_balance` ≥ 0 are positive integers.
  - Checks `cust_pk` is a valid Tezos EdDSA public key for the curve specified by `tezos-client` and that `customer_address` is a valid Tezos tz1 address that is correctly derived from `cust_pk`.
  - Checks `merch_pp_hash` is the SHA3-256 hash of` (merchant_zkabacus_public_key, merchant_address, merch_pk)`.
  - Checks `customer_address` is an implicit Tezos account (tz1 address), and not a smart contract address (KT1 address). 

The merchant may choose to either accept or reject the channel establishment request. If the merchant accepts, they should ensure their implicit Tezos account with address `merchant_address` has a balance sufficient to both contribute the desired amount to the zkChannel and pay the [operations fees](5-tezos-escrowagent.md#operation-fees) needed to fund and call the appropriate entry points of the corresponding smart contract. We recommend 0.009 tez based on our [contract benchmarks](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results) on testnet.

### The `open_m` Message
The merchant sends the `open_m` message to the customer; this message is formed as follows:
1. type: `open_m`
2. data: [`string`:`channel_id_m`]. This is the merchant randomness contribution to the channel identifier.

#### Customer Requirements
Upon receipt, the customer checks the that `channel_id_m` is a valid merchant randomness contribution to the channel identifier. If so, the customer sets the channel identifier `channel_id` to: `SHA3-256(channel_id_c, channel_id_m, cust_pk, merch_pk, merchant_zkabacus_public_key)`, where:
- `channel_id_c` is the customer randomness contribution to the channel identifier sent to the merchant in the `open_c` message.
- `channel_id_m` is the merchant randomness contribution to the channel identifier received in the `open_m` message.
- `cust_pk` is the customer Tezos account public key.
- `merch_pk` is the merchant Tezos account public key.
- `merchant_zkabacus_public_key` is the merchant's zkAbacus Pointcheval Sanders public key.

If not, the customer aborts.

#### Merchant Requirements
Before sending, the merchant:
  - Generates `channel_id_m` randomly using a secure RNG.
  - Sets the channel identifier `channel_id` to: `SHA3-256(channel_id_c, channel_id_m, cust_pk, merch_pk, merchant_zkabacus_public_key)`, where:
    * `channel_id_c` is the customer randomness contribution to the channel identifier sent to the merchant in the `open_c` message.
    * `channel_id_m` is the merchant randomness contribution to the channel identifier received in the `open_m` message.
    * `cust_pk` is the customer Tezos account public key.
    * `merch_pk` is the merchant Tezos account public key.
    * `merchant_zkabacus_public_key` is the merchant's zkAbacus Pointcheval Sanders public key.


### The `init_c` Message
The customer sends an `init_c` message to the merchant.

1. type: `init_c`
2. data:
    * [`string`:`channel_id`]
    * [`bls12_381_g1`:`close_state_commitment`]: A commitment to the initial closing state.
    * [`bls12_381_g1`:`state_commitment`]: A commitment to the initial state.
    * [`(bls12_381_g1, bls12_381_g1, Vec<bls12_381_fr>): establish_proof`]: A zero-knowledge proof of correctness of the commitments to the initial state and initial closing state.

#### Customer Requirements
The customer runs the `zkAbacus.Initialize()` on inputs `channel_id`, `customer_balance`, and `merchant_balance` to generate the `init_c` message.

#### Merchant Requirements
Upon receipt, the merchant:
- Checks that `channel_id` matches the channel identifier previously computed.
- Continues as specified in `zkAbacus.Initialize()`. 

If `channel_id` is incorrect, the merchant aborts.

### The `init_m` Message
The merchant sends an `init_m` message to the customer.

1. type: `init_m`
2. data: [`(bls12_381_g1, bls12_381_g1):closing_signature`]. A closing authorization signature on the initial closing state.
 
#### Customer Requirements
Upon receipt, the customer verifies `closing_signature` is a valid signature with respect to the merchant zkAbacus Pointcheval Sanders public key. If the signature is valid, the customer continues as specified in `zkAbacus.Initialize()`. If the signaure is invalid, the customer aborts.

#### Merchant Requirements
The merchant runs `zkAbacus.Initialize` on inputs `channel_id`, `customer_balance`, and `merchant_balance`. If successful, the merchant sends the resulting message `init_m`. Otherwise, the merchant aborts.

### The `funding_confirmed` Message
The customer sends the `funding_confirmed` message to the merchant.

1. type: `funding_confirmed`
2. data: 
    * [`address`:`contract-id`]: The contract identifier for the zkChannels smart contract.
    * [`int`:`originated-block-height`]: The block height of the zkChannels smart contract.

#### Customer Requirements

The customer:
  - Originates the contract as specified in the [Tezos zkEscrowAgent Realization document](5-tezos-escrowagent.md#contract-origination-and-funding).
  - Waits until the origination operation is confirmed on chain for at least `required_confirmations` blocks. 
  - Updates the channel status to `Originated`.
  - Funds the contract using [the `addFunding` entrypoint](5-tezos-escrow-agent#addfunding).
  - Waits until the funding operation is confirmed on chain for at least `required_confirmations` blocks. 
  - Updates the channel status to `CustomerFunded`.
  - Sends the `funding_confirmed` message to the merchant.

#### Merchant Requirements
Upon receipt of the `funding_confirmed` message, the merchant: 
  - Checks that the originated contract `contract-id` contains the expected [zkchannels contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz) with respect to the channel identifier `channel_id`, the customer Tezos public key `cust_pk`, the customer's tezos tz1 address `customer_address`, the merchant public parameters, and the initial balances `customer_balance` and `merchant_balance`.
  - Checks that the on-chain storage of `contract-id` at `originated-block-height` is exactly as expected for channel `channel_id`:
    - The contract storage contains the merchant's Pointcheval Sanders public key.
    - The customer's tezos tz1 address and public key match the fields `customer_address` and  `cust_pk`, respectively.
    - The merchant's tezos tz1 address and public key match the fields `merchant_address` and  `merch_pk`, respectively.
    - The `self_delay` field in the contract matches the value specified in the [global defaults](1-setup.md#Global-defaults). 
    - The `close` field in the contract matches the merchant's `close` flag defined as defined in the [global defaults](1-setup.md#Global-defaults). The `close` flag represents a fixed scalar used by the merchant to differentiate closing state and state.
    - `custFunding` and `merchFunding` match the initial balances `customer_balance` and `merchant_balance`, respectively.
    - The `status` field of the contract is set to `0`, which corresponds to `AWAITING_FUNDING`.
    - The `context-string` is set to `"zkChannels mutual close"`, as defined in the [global defaults](1-setup.md#Global-defaults). 
  - Waits until the originated contract is confirmed on chain for at least `required_confirmation` blocks.
  - Updates the channel status to `Originated`.
  - Checks that the customer has funded the contract and waits until the customer's side of the contract has been funded for at least `required_confirmations` blocks. This requires checking that the customer's operation to add their funds is the last operation to have interacted with the smart contract, and that in the most recent blocks of the blockchain (up to `required_confirmations` blocks in the past) there have been no further operations interacting with the contract. 
  - In the dual-funded case, funds their side of the contract by calling the `addFunding` entrypoint as specified in [Contract Origination and Funding](5-tezos-escrowagent.md#contract-origination-and-funding). 
  - Waits until the `addFunding` operation is confirmed on chain for at least `required_confirmation` blocks.
  - Updates the channel status to `MerchantFunded`.

  ### The `activate` Message
  The merchant sends the `activate` message to the customer.
  
1. type: `activate`
2. data: [(bls12_381_g1, bls12_381_g1):`payment_tag`]

#### Customer Requirements
Upon receipt, the customer:
  - In the dual-funded case, waits for the contract storage status to be `OPEN` for `required_confirmations` blocks before proceeding. Update the channel status to `MerchantFunded`.
  - If the customer does not see a confirmed `addFunding` operation from the merchant within a specified timeout period, they call [the `reclaimFunding` entrypoint](5-tezos-escrowagent#reclaimfunding).
  - Checks that `payment_tag` is a valid signature with respect to the merchant's zkAbacus Pointcheval Sanders public key. If not, aborts by initiating a unilateral close.
  - Updates the channel status to `Ready`.

#### Merchant Requirements
Before sending, the merchant:
  - Waits until the contract storage status has been set to `OPEN` (denoted as `1`) for `required_confirmations` blocks.
  - Generate the `activate` message by running `zkAbacus.Activate()` on the initial state commitment `state_commitment` provided in the customer's `init_c` message, the channel identifier `channel_id`, and their Pointcheval Sanders public key.
  - Updates the channel status to `Active`.