# Channel Establishment
  * [Prerequisites](#prerequisites)
  * [Protocol Overview](#protocol-overview)
  * [Message Specifications](#message-specifications)
    * [The `open_c` Message](#the-open_c-message)
    * [The `open_m` Message](#the-open_m-message)
    * [The `init_c` Message](#the-init_c-message)
    * [The `init_m` Message](#the-init_m-message)
    * [The `funding_confirmed` Message](#the-funding_confirmed-message)
    * [The `activate` Message](#the-activate-message)

Channel establishment is initiated by the customer in zkChannels. 

## Prerequisites
The merchant has completed the [setup](1-setup.md#merchant-setup) phase, and the customer and merchant have established a communication session.

The customer has [obtained the merchant’s setup information](1-setup.md#publishing-public-parameters) out of band. The customer must verify the merchant's public parameters are well-formed and valid:
* The merchant blind signing public key `merchant_zkabacus_public_key` must consist of a valid Pointcheval Sanders public key of the expected length with components in the BLS12-381 pairing subgroups G1 and G2.
* The range constraint parameters `range_constraint_parameters` must consist of a valid Pointcheval Sanders key of the expected length with components in the BLS12-381 pairing subgroup G1, and valid signatures on the appropriate integer range.
* The revocation lock commitment parameters `revocation_commitment_parameters` must be well-formed Pedersen parameters of the expected length, and consist of elements in the BLS12-381 pairing subgroup G1.
* The merchant EdDSA public key `merchant_public_key` must be a valid EdDSA key for the curve specified by `tezos-client` and the merchant address `merchant_address` must be a Tezos tz1 address correctly derived from `merchant_public_key`. 

The customer should ensure they have a Tezos implicit account with balance sufficient to both contribute the desired amount to the zkChannel and pay the [operations fees](5-tezos-escrowagent.md#operation-fees) needed to originate, fund, and call the appropriate entry points of the corresponding smart contract. We recommend 2 tez based on our [contract benchmarks](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results) on testnet.

## Protocol Overview


Channel establishment is a three-round protocol between the customer and the merchant.

The establishment protocol uses `zkAbacus` as a subprotocol.  Details for `zkAbacus` may be found in Chapter 3.3.3 of the [zkChannels Protocol document](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review-v1/zkchannels-protocol-spec-v3.1.pdf). Each party also interacts with the Tezos blockchain to open and verify the channel's escrow account. Details of on-chain operations are provided [here](5-tezos-escrowagent.md).

The protocol flows between the customer and the merchant are given in the following diagram:


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

The protocol proceeds as follows:
        
1. The customer sends the [`open_c` message](#the-open_c-message), which contains information about the initial state of the proposed channel. 
2. The merchant [verifies the received message](#merchant-requirements) and either accepts or rejects the proposed channel. They reply with [the `open_m` message](#the-open_m-message), which contains the merchant's contribution to the channel identifier. 
3. The customer and merchant each compute the channel identifer `channel_id`, which acts as the unique channel identifier for the on-chain Tezos escrow account and off-chain `zkAbacus` channel. The identifier `channel_id` is computed as `SHA3-256(customer_randomness, merchant_randomness, customer_public_key, merchant_public_key, merchant_zkabacus_public_key)`. 
4. They then initialize the `zkAbacus` channel by running `zkAbacus.Initialize()` on the previously established public parameters. In this subroutine:
    
   a. The customer [sends the `init_c` message](#the-init_c-message) to the merchant. This message consists of a (hiding) commitment to the intial state and a zero-knowledge proof of correctness. 

    b. The merchant [verifies the received message and sends the `init_m` message](#the-init_m-message), which contains an initial closing authorization signature, to the customer. 

5. The customer originates and funds the [zkChannels contract](5-tezos-escrowagent#zkchannels-contract) on chain:
    
    a.  They forge and sign the [origination operation](5-tezos-escrowagent.md#zkchannels-contract-origination-operation) with the following arguments:        
      * `channel_id`: The channel identifier.
      * `customer_address`: The customer's Tezos tz1 address.
      * `init_customer_balance`: The customer's initial balance.
      * `customer_public_key`: The customer's Tezos public key.
      * `merchant_address`: The merchant's Tezos tz1 address.
      * `init_merchant_balance`: The merchant's initial balance.
      * `merchant_public_key`: The merchant's Tezos public key.
      * `merchant_zkabacus_public_key`: The merchant's zkAbacus public key.

      b.  They inject the [origination operation](5-tezos-escrowagent.md#zkchannels-contract-origination-operation). They wait until this operation is confirmed to depth `required_confirmations` and then they update their channel status to `Originated`.

      c.  They fund the contract by [calling the `addCustFunding` entrypoint](5-tezos-escrowagent.md#addcustfunding-entrypoint) of the contract. The source of this transfer operation must be the `customer_address` specified in the contract's initial storage and transfer amount must be equal to `init_customer_balance`. They wait until the `addCustFunding` operation group is confirmed to the depth `required_confirmations` and then update the channel status to `CustomerFunded`.
      
6. The customer [sends the `funding_confirmed` message](#the-funding_confirmed-message) to the merchant, which contains the contract identifier `contract-id`.
7. The merchant [checks the corresponding contract and initial storage for the expected values](#merchant-requirements-4). The merchant then funds their side of the smart contract, if the channel is dual-funded, by [calling the `AddMerchFunding` entrypoint](5-tezos-escrowagent.md#addmerchfunding-entrypoint) of the contract with identifier `contract_id`. Once this contract has status `OPEN` for `required_confirmations` blocks, the merchant runs `zkAbacus.Activate()` to generate the initial payment tag and [sends the customer the message `activate`](#the-activate-message), which contains this payment tag. 
8. Upon completion of `zkAbacus.Activate()`, the channel is open and ready for [payments](3-channel-payments.md). 


## Message Specifications

### The `open_c` Message
The `open_c` message is sent from the customer to the merchant and is formed as follows:

1. type: `open_c`
2. data: 
    * [`string`:`customer_randomness`]: Customer randomness contribution to the channel identifer.
    * [`int`:`init_customer_balance`]: The proposed initial customer balance.
    * [`int`:`init_merchant_balance`]: The proposed initial merchant balance.
    * [`address`:`customer_address`]: The customer's Tezos tz1 account address.
    * [`key`:`customer_public_key`]: The customer's Tezos EdDSA public key.
    * [`string`: `merch_pp_hash`]: A hash of merchant public parameters as specified in [Merchant Setup](1-setup.md#merchant-setup).

#### Customer Requirements

The customer, before sending:
- Retrieves the merchant public parameters and checks these parameters are well-formed and valid as specified [above](#prerequisites).
- Generates `customer_randomness` uniformly at random using a secure RNG. 


#### Merchant Requirements

Upon receipt, the merchant checks that the following are true. If any are false, the merchant aborts:
  - Checks `customer_randomness` is the correct length.
  - Checks `init_customer_balance` ≥ 0 and `init_merchant_balance` ≥ 0 are positive integers.
  - Checks `customer_public_key` is a valid Tezos EdDSA public key for the curve specified by `tezos-client` and that `customer_address` is a valid Tezos tz1 address that is correctly derived from `customer_public_key`.
  - Checks `merch_pp_hash` is the SHA3-256 hash of` (merchant_zkabacus_public_key, merchant_address, merchant_public_key)`.
  - Checks `customer_address` is an implicit Tezos account (tz1 address), and not a smart contract address (KT1 address). 

The merchant may choose to either accept or reject the channel establishment request. The implementation should provide a customizable approver mechanism in order to realize channel establishment approvals and rejections. If the merchant accepts, they should ensure their implicit Tezos account with address `merchant_address` has a balance sufficient to both contribute the desired amount to the zkChannel and pay the [operations fees](5-tezos-escrowagent.md#operation-fees) needed to fund and call the appropriate entry points of the corresponding smart contract. We recommend 0.009 tez based on our [contract benchmarks](https://github.com/boltlabs-inc/tezos-contract/wiki/Benchmark-Results) on testnet.

### The `open_m` Message
The merchant sends the `open_m` message to the customer; this message is formed as follows:
1. type: `open_m`
2. data: [`string`:`merchant_randomness`]. This is the merchant randomness contribution to the channel identifier.

#### Customer Requirements
Upon receipt, the customer checks the that `merchant_randomness` is the correct length. If so, the customer sets the channel identifier `channel_id` to: `SHA3-256(customer_randomness, merchant_randomness, customer_public_key, merchant_public_key, merchant_zkabacus_public_key)`, where:
- `customer_randomness` is the customer's contribution to the channel identifier sent to the merchant in the `open_c` message.
- `merchant_randomness` is the merchant's contribution to the channel identifier received in the `open_m` message.
- `customer_public_key` is the customer Tezos account public key.
- `merchant_public_key` is the merchant Tezos account public key.
- `merchant_zkabacus_public_key` is the merchant's zkAbacus Pointcheval Sanders public key.

If not, the customer aborts.

#### Merchant Requirements
Before sending, the merchant:
  - Generates `merchant_randomness` uniformly at random using a secure RNG.
  - Sets the channel identifier `channel_id` to: `SHA3-256(customer_randomness, merchant_randomness, customer_public_key, merchant_public_key, merchant_zkabacus_public_key)`, where:
    * `customer_randomness` is the customer's contribution to the channel identifier sent to the merchant in the `open_c` message.
    * `merchant_randomness` is the merchant's contribution to the channel identifier received in the `open_m` message.
    * `customer_public_key` is the customer Tezos account public key.
    * `merchant_public_key` is the merchant Tezos account public key.
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
The customer runs the `zkAbacus.Initialize()` on inputs `channel_id`, `init_customer_balance`, and `init_merchant_balance` to generate the `init_c` message.

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
The merchant runs `zkAbacus.Initialize` on inputs `channel_id`, `init_customer_balance`, and `init_merchant_balance`. If successful, the merchant sends the resulting message `init_m`. Otherwise, the merchant aborts.

### The `funding_confirmed` Message
The customer sends the `funding_confirmed` message to the merchant.

1. type: `funding_confirmed`
2. data: 
    * [`address`:`contract-id`]: The contract identifier for the zkChannels smart contract.
    * [`int`:`originated-block-height`]: The block height of the zkChannels smart contract.

#### Customer Requirements

The customer:
  - Originates and funds the contract as specified in the [Tezos zkEscrowAgent Realization document](5-tezos-escrowagent.md#zkchannels-customer-origination-and-funding-protocol).
  - Waits until the origination operation is confirmed on chain for at least `required_confirmations` blocks. 
  - Updates the channel status to `Originated`.
  - Funds the contract by [calling the `addCustomerFunding` entrypoint](5-tezos-escrow-agent#addCustFunding-entrypoint).
  - Waits until the funding operation is confirmed on chain for at least `required_confirmations` blocks. 
  - Updates the channel status to `CustomerFunded`.
  - Sends the `funding_confirmed` message to the merchant.

#### Merchant Requirements
Upon receipt of the `funding_confirmed` message, the merchant: 
  - Checks that the originated contract `contract-id` contains the expected [zkchannels contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz) with respect to the channel identifier `channel_id`, the customer Tezos public key `customer_public_key`, the customer's tezos tz1 address `customer_address`, the merchant public parameters, and the initial balances `init_customer_balance` and `init_merchant_balance`.
  - Checks that the on-chain storage of `contract-id` at `originated-block-height` is exactly as expected for channel `channel_id`:
    - The contract storage contains `merchant_zkabacus_public_key` in the expected field(s).
    - The customer's tezos tz1 address and public key match the fields `customer_address` and  `customer_public_key`, respectively.
    - The merchant's tezos tz1 address and public key match the fields `merchant_address` and  `merchant_public_key`, respectively.
    - The `self_delay` field in the contract matches the value specified in the [global defaults](1-setup.md#Global-defaults). 
    - The `close` field in the contract matches the merchant's `close` flag defined as defined in the [global defaults](1-setup.md#Global-defaults). The `close` flag represents a fixed scalar used by the merchant to differentiate closing state and state.
    - The fields `customer_balance` and `merchant_balance` are initialized to `init_customer_balance` and `init_merchant_balance`, respectively.
    - The `status` field of the contract is initialized to `AWAITING_CUST_FUNDING`.
    - The `context-string` is set to `"zkChannels mutual close"`, as defined in the [global defaults](1-setup.md#Global-defaults). 
  - Waits until the originated contract is confirmed on chain for at least `required_confirmations` blocks.
  - Updates the channel status to `Originated`.
  - Checks that the customer has funded the contract and waits until the customer's side of the contract has been funded for at least `required_confirmations` blocks. This requires checking that the customer's operation to add their funds is the last operation to have interacted with the smart contract, and that in the most recent blocks of the blockchain (up to `required_confirmations` blocks in the past) there have been no further operations interacting with the contract. 
  - In the dual-funded case, funds their side of the contract by [calling the `addMerchantFunding` entrypoint](#addmerchmunding-entrypoint). The source of this transfer operation must be the `merchant_address` specified in the contract's initial storage and the transfer amount must be equal to `init_merchant_balance`.
  - Waits until the `addMerchantFunding` operation is confirmed on chain and the contract storage `status` is `OPEN` for at least `required_confirmations` blocks.
  - Updates the channel status to `MerchantFunded`.

  ### The `activate` Message
  The merchant sends the `activate` message to the customer.
  
1. type: `activate`
2. data: [(bls12_381_g1, bls12_381_g1):`payment_tag`]

#### Customer Requirements
Upon receipt, the customer:
  - In the dual-funded case, waits for the contract storage status to be `OPEN` for `required_confirmations` blocks before proceeding. Update the channel status to `MerchantFunded`.
  - If the customer does not see a confirmed `addMerchantFunding` operation from the merchant within a specified timeout period, they call [the `reclaimFunding` entrypoint](5-tezos-escrowagent#reclaimfunding).
  - Checks that `payment_tag` is a valid signature with respect to the merchant's zkAbacus Pointcheval Sanders public key. If not, aborts by initiating a unilateral close.
  - Updates the channel status to `Ready`.

#### Merchant Requirements
Before sending, the merchant:
  - Waits until the contract storage status has been set to `OPEN` for `required_confirmations` blocks.
  - Generate the `activate` message by running `zkAbacus.Activate()` on the initial state commitment `state_commitment` provided in the customer's `init_c` message, the channel identifier `channel_id`, and their Pointcheval Sanders public key.
  - Updates the channel status to `Active`.
