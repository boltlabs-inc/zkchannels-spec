# Tezos on-chain operations
* [Introduction](#introduction)
* [Operation structure](#operation-structure)
* [Operation signature](#operation-signature)
* [zkChannel operations](#zkchannel-operations)
    * [Contract origination](#contract-origination)
    * [Contract calls](#contract-calls)
        * [`addFunding`](#`addfunding`)
        * [`reclaimFunding`](#`reclaimFunding`)
        * [`expiry`](#`expiry`)
        * [`custClose`](#`custClose`)
        * [`merchDispute`](#`merchDispute`)
        * [`custClaim`](#`custClaim`)
        * [`merchClaim`](#`merchClaim`)
        * [`mutualClose`](#`mutualClose`)

## Introduction
In order to use zkChannels, the customer and merchant need to perform some on-chain operations such as originating the smart contract or calling its entrypoints. Creating and broadcasting these operations generally involves interacting with a Tezos node. Below are example commands for performing the zkChannel operations using the [Tezos client](https://gitlab.com/tezos/tezos.git) implementation.

[Here](https://edo2net.tzkt.io/KT1FVYH6bbfYxYntiESPmMevjcT9Co3nEYpM/operations/) is an example of the zkChannel smart contract on testnet where you can view also operations that have called it.

## Operation structure

Below is an example of an unsigned tezos operation. The signature on this operation is created by serializing the operation and signing it with the pivate key associated with the `source` address. For detailed description of how Tezos operations get serialized and signed see [this post](https://www.ocamlpro.com/2018/11/21/an-introduction-to-tezos-rpcs-signing-operations/).
```
{
  "branch": "BMHBtAaUv59LipV1czwZ5iQkxEktPJDE7A9sYXPkPeRzbBasNY8",
  "contents": [
    { "kind": "transaction",
      "source": "tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51",
      "fee": "50000",
      "counter": "3",
      "gas_limit": "20000",
      "storage_limit": "0",
      "amount": "0",
      "destination": "KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa",
      "parameters": {
        "entrypoint": <entrypoint>,
        "args": <arguments>
      }
    }
}
```
- `branch`: A block hash of a recent block in the Tezos blockchain. If the block hash is deeper than 60 blocks, the operation will become invalid.
- `kind`: The type of operation. Origination have type `"origination"` and contract calls and regular transfers have type `"transaction"`.
- `source`: The sender address.
- `fee`: The fee that goes to the baker (in mutez).
- `counter`: Unique sender account ‘nonce’ value. It is used to prevent replay attacks.
- `gas_limit`: Caller-defined gas limit. 
- `storage_limit`: Caller-defined storage limit. 
- `amount`: The value being transferred (in mutez).
- `destination`: The destination address.
- `parameters`: Contains additional parameters used when interacting with smart contracts.
- `entrypoint`: The entrypoint of the smart contract being called.
- `args`: the arguments to be passed in to the entrypoint.

The `fee`, `storage_limit`, and `gas_limit` are to be determined by the user (the customer or merchant) broadcasting their operation. The only requirement as far as the protocol is concerned is that the fees and limits are sufficient for the operation to be baked. The tezos client used to interact with the blockchain should be capable of calculating the gas and storage used by an operation, as well as a fee that will ensure it can get baked in a timely manner with a high probability. 
## Operation signature
The operation is translated into a binary format, e.g.
```
ce69c5713dac3537254e7be59759cf59c15abd530d10501ccf9028a5786314cf08000002298c03ed7d454a101eb7022bc95f7e5f41ac78d0860303c8010080c2d72f0000e7670f32038107a59a2b9cfefae36ea21f5aa63c00
```
The prefix `03` is added to the bytes. `03` is a watermark for operations in Tezos.

```
03ce69c5713dac3537254e7be59759cf59c15abd530d10501ccf9028a5786314cf08000002298c03ed7d454a101eb7022bc95f7e5f41ac78d0860303c8010080c2d72f0000e7670f32038107a59a2b9cfefae36ea21f5aa63c00
```
The operation is then hashed using blake2b to create a 32-byte digest. The digest is then signed using the private key of the `source` address using the EdDSA on the Ed25519 curve.

```
637e08251cae646a42e6eb8bea86ece5256cf777c52bc474b73ec476ee1d70e84c6ba21276d41bc212e4d878615f4a31323d39959e07539bc066b84174a8ff0d
```
In the final step, the binary format of the operation is appended to the signature to create the signed operation which is ready to be broadcast by the Tezos node.


## zkChannel operations
### Contract origination operations
The contract origination operation has the following format:

```
"kind": "origination",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"balance": "0",
"script": {
    "code": <zkchannel_contract>,
    "storage": <initial_storage> 
}
```
`<cust_addr>` is the customer's tz1 address and should match the `cust_addr` field in the contract initial storage. This account will also be used to fund the channel. `<zkchannel_contract>` The [zkchannels contract](https://github.com/boltlabs-inc/tezos-contract/blob/main/zkchannels-contract/zkchannel_contract.tz) in michelson. `<initial_storage>` contains the initial storage parameters as listed in [2-contract-origination.md](2-contract-origination.md#initial-storage-arguments). The expected format of the storage is:

```
(Pair 
    (Pair 
        (Pair 
            (Pair <cid> <close>) 
            (Pair 
                <context_string> 
                (Pair <cust_addr> <custBal>)
            )
        ) 
        (Pair 
            (Pair 
                <cust_funding> 
                (Pair <cust_pk> <delayExpiry>)
            ) 
            (Pair 
                <g2> 
                (Pair <merch_addr> <merchBal>)
            )
        )
    ) 
    (Pair 
        (Pair 
            (Pair <merch_funding> <merch_pk>) 
            (Pair 
                <merchPk0> 
                (Pair <merchPk1> <merchPk2>)
            )
        ) 
        (Pair 
            (Pair 
                <merchPk3> 
                (Pair <merchPk4> <merchPk5>)
            ) 
            (Pair 
                <rev_lock> 
                (Pair <self_delay> <status>)
            )
        )
    )
)
```

### Contract calls
Contract calls are transfer operations where the destination is the smart contract. Entrypoints can be specified in the transaction and any additional arguments can be included with the `--arg` flag, `"Unit"` is the default value for when there are no additional arguments. 

In the following operations, `<contract_id>`  is the contract ID (KT1 address) of the zkChannel contract.


#### `addFunding`
```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": <cust_funding>,
"destination": <contract_id>,
"parameters": {
  "entrypoint": "addFunding",
  "args": "Unit"
}
```
`<cust_funding>` specifies the amount to be transferred to fund the contract in mutez. This amount must be exactly equal to the `cust_funding` specified in the contract's initial storage.
#### `reclaimFunding`

```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "reclaimFunding",
  "args": "Unit"
}
```
#### `expiry`
```
"kind": "transaction",
"source": <merch_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "expiry",
  "args": "Unit"
}
```

#### `custClose`
```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": <cust_funding>,
"destination": <contract_id>,
"parameters": {
  "entrypoint": "custClose",
  "args": 
}
```
The arguments passed into `custClose` are: 
- the closing balances, `cust_bal` and `merch_bal`, 
- the revocation lock for that state, `rev_lock`,
- and the merchant's closing authorization signatures, `s1` and `s2`.

The expected format for `<cust_close_storage>` is:
```
(Pair 
    (Pair <cust_bal> <merch_bal>) 
    (Pair 
        <rev_lock> 
        (Pair <s1> <s2>))
)
```
#### `merchDispute`
```
"kind": "transaction",
"source": <merch_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "merchDispute",
  "args": <revocation_secret>
}
```
`<revocation_secret>` is the SHA3 preimage for `rev_lock` that had been broadcasted by the customer as part of the `custClose` entrypoint call.
#### `custClaim`
```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "custClaim",
  "args": "Unit"
}
```
#### `merchClaim`
```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "merchClaim",
  "args": "Unit"
}
```
#### `mutualClose`
```
"kind": "transaction",
"source": <cust_addr>,
"fee": <fee>,
"counter": <counter>,
"gas_limit": <gas_limit>,
"storage_limit": <storage_limit>,
"amount": "0",
"destination": <contract_id>,
"parameters": {
  "entrypoint": "mutualClose",
  "args": <mutual_close_storage>
}
```
`<mutual_close_storage>` contains the closing balances that have been agreed upon by the customer and merchant, `cust_bal` and `merch_bal`, and the merchant's EdDSA signature, `merch_sig`. The expected format for `<mutual_close_storage>` is:
```
(Pair 
    <cust_bal> 
    (Pair <merch_bal> <merch_sig>)
)
```