# Tezos client operations
* [Introduction](#introduction)
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

## Contract origination
```
originate contract my_zkchannel transferring 0 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 running zkchannel_contract.tz --init <initial_storage>
```
- `my_zkchannel` is an alias the node can use to refer to the contract.
- `tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51` is the customer's address 
- `<initial_storage>` would contain the initial storage parameters as listed in [2-contract-origination.md](2-contract-origination.md#initial-storage-arguments)


## Contract calls

Contract calls are 0 value transfer operations where the destination is the smart contract. Entrypoints can be specified in the transaction and any additional arguments can be included with the `--arg` flag, `'Unit'` is the default value for when there are no additional arguments. 

In the following examples, 
- `KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa` : smart contract address
- `tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51` : customer's address 
- `tz1VcYZwxQoyxfjhpNiRkdCUe5rzs53LMev6` : merchant's address


### `addFunding`
```
transfer 10 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'addFunding' --arg 'Unit'
```

### `reclaimFunding`
```
transfer 0 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 to KT1AxxzbDAk8k3eTbLidgbkoukt6RTMFFVuf --entrypoint 'reclaimFunding' --arg 'Unit'
```

### `expiry`
```
transfer 0 from tz1VcYZwxQoyxfjhpNiRkdCUe5rzs53LMev6 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'expiry' --arg 'Unit'
```

### `custClose`
```
transfer 0 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'custClose' --arg 'Pair (Pair 5000000 25000000) 0x50dc0701ca265bc9dcec0e8227932edf2f67ab8e10a199830f10963da1f0772a 0x12793e9a17f581a7c4ab23aae1bb63d8b2a4c743e0d84eb655ce5f984e4c6b70efa82795360c3231fd99e1a750ff93161050944b741b8e417da09409dc67a12da50505a65b37885cfd6993cd11f98312ac283b0b8b25526278572e21a77c98e4 0x0a4d50d372cd83c7d4bb32fe2e009f2534fcedec655690df53d05dfa280d76067b4fec53804f7e784503aeefd2d84e1006db47b5292042d433cef04c2494a484702f566a456315308d82621b3bf8f95a3d0bd10c1f9b91c4bab18b341648fadb'
```
Arguments
- `5000000` : customer closing balance
- `25000000` : merchant closing balance
- `0x50dc0701ca265bc9dcec0e8227932edf2f67ab8e10a199830f10963da1f0772a` : revocation lock
- `0x12793e9a17f581a7c4ab23aae1bb63d8b2a4c743e0d84eb655ce5f984e4c6b70efa82795360c3231fd99e1a750ff93161050944b741b8e417da09409dc67a12da50505a65b37885cfd6993cd11f98312ac283b0b8b25526278572e21a77c98e4` : merchant's closing authorization siganture (s1)
- `0x0a4d50d372cd83c7d4bb32fe2e009f2534fcedec655690df53d05dfa280d76067b4fec53804f7e784503aeefd2d84e1006db47b5292042d433cef04c2494a484702f566a456315308d82621b3bf8f95a3d0bd10c1f9b91c4bab18b341648fadb` : merchant's closing authorization siganture (s2)

### `merchDispute`
```
transfer 0 from tz1VcYZwxQoyxfjhpNiRkdCUe5rzs53LMev6 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'merchDispute' --arg '0x1f0178304fea6045e851ca6d1bb613a093ea4e0e6f92010c44473a56807993ed'
```
Arguments
- `0x1f0178304fea6045e851ca6d1bb613a093ea4e0e6f92010c44473a56807993ed` : revocation secret
### `custClaim`
```
transfer 0 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'custClaim' --arg 'Unit'
```

### `mutualClose`
```
transfer 0 from tz1S6eSPZVQzHyPF2bRKhSKZhDZZSikB3e51 to KT1EHTCw75YWw77HffBJgArxugfeXrTZibSa --entrypoint 'mutualClose' --arg (Pair 5000000 (Pair 25000000 "edsigtb8SMsNQQYaRkvzvHxZhY2eNVHBoUieZ3PykEEm4Arj47YAqPPP9iqiDH54bxBxprXQMVjVyUuN9GRWqapxiSopWGq4M1t"))
```
Arguments
- `5000000` : customer closing balance
- `25000000` : merchant closing balance
- `"edsigtb8SMsNQQYaRkvzvHxZhY2eNVHBoUieZ3PykEEm4Arj47YAqPPP9iqiDH54bxBxprXQMVjVyUuN9GRWqapxiSopWGq4M1t"` : merchant's mutual close signature