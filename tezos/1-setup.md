# Setup
  * [System Setup](#system-setup)
    * [Commitment scheme parameters](#commitment-scheme-parameters)
    * [Signature schemes](#signature-schemes)
    * [Global defaults](#global-defaults)
  * [Merchant Setup](#merchant-setup)
    * [Blind signing key generation](#blind-signing-key-generation)
    * [Range proof parameters generation](#range-proof-parameters-generation)
    * [EdDSA and Tezos address generation](#eddsa-and-tezos-address-generation)
    * [Publishing public parameters](#publishing-public-parameters)
    * [Tezos-related node initialization](#tezos-related-node-initialization)
    * [Revocation database initialization](#revocation-database-initialization)

## System Setup
### Commitment scheme parameters
* **Hash commitment parameters**:
We use SHA3-256 hashes to instantiate our hash-based commitments.

### Signature schemes
* Pointcheval-Sanders signatures public parameters: We use the pairing defined by the [BLS12-381](https://crates.io/crates/bls12_381) crate. 
* EdDSA signature public parameters: We rely on the public parameters selected by the `tezos-client` for Ed25519.

### Global defaults
#### `TezosEscrowAgent` global defaults
* `self_delay`: an integer that represents the length of the timeout period. The same delay is applied to the `expiry` and `custClose` entrypoints. The value is interpreted in seconds. The default value is set to 172800, based on the number of seconds in 48 hours.
* `required_confirmations`: an integer that represents the minimum number of confirmations for an operation to be considered final. The default value is set to 20.
* `context-string`: a string set to `"zkChannels mutual close"`. This is contained in the tuple that gets signed when creating `mutual_close_signature`.
#### `zkAbacus` global defaults
* `close`: a fixed scalar used to differentiate closing state and state. The default value is 0x000000000000000000000000000000000000000000000000000000434c4f5345, which is derived from the binary encoding of the string 'CLOSE'.

## Merchant Setup

A party who wishes to act as a zkChannel merchant must generate and publish the following parameters for use across all of their zkChannels: 

* **Blind signature public key pair.** The merchant must use this key pair to sign all channel state initializations and updates in `zkAbacus`. This keypair is also used as a condition in the Tezos smart contract for each channel. 

* **Range proof public key pair and signatures.** The `zkAbacus.Pay` protocol requires a separate set of parameters to prove that updated channel balances are valid. A range proof demonstrates that a value (in our case, an updated channel balance) is non-negative. Specifically, we use range proofs to show that an integer is in the range `[0, u^l)`, for integer values `u`, `l`. We set `u = 128` and `l = 9`. This primitive relies on Poincheval Sanders signatures.

* **Revocation lock commitment parameters.** The `zkAbacus.Pay` protocol requires a set of Pedersen commitment parameters on messages of length one in the pairing group `G1` of BLS12-381. These are used to prove that the customer reveals the correct revocation lock.

* **EdDSA public key pair and Tezos tz1 address.** The merchant uses this keypair for escrow account set up and closing as specified in `TezosEscrowAgent`.


The merchant must also initialize a database `revocation_DB` for use with all their zkChannel operations.

The merchant completes setup as follows.
### Blind signing key generation

Generate a [new Pointcheval-Sanders keypair](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) for signatures on message tuples of length five.
This key pair must be used across all channels with the merchant.

### Range proof parameters generation

Generate a [new Pointcheval-Sanders keypair](
https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/zkchannels-crypto/src/pointcheval_sanders.rs#L255) for signatures on message tuples of length one.
With this keypair, separately [sign](
https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/zkchannels-crypto/src/pointcheval_sanders.rs#L271) each integer in the range `[0, u-1]`.

The resulting public key and signatures form the the merchant's range proof parameters. This keypair must not be used for any other purpose.

### Revocation lock commitment parameters generation

Generate a new set of [Pedersen commitment parameters](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/zkchannels-crypto/src/pedersen.rs#L85) for commitments to tuples of length one. 

### EdDSA and Tezos address generation
Generate an EdDSA keypair and associated Tezos tz1 address using the `tezos-client`.

### Publishing public parameters
The merchant's public parameters consists of their blind signing public key `merch_PS_pk`, range proof parameters `range_proof_params`, revocation commitment parameters `revlock_com_params`, EdDSA public key `merch_pk`, and Tezos tz1 address `merch_addr`. 

The merchant publishes their public parameters in a config file. The merchant then advertises their server IP address and port for customers to open channels using the `<merch_pp_hash>@<ip>:<port>` format, where `merch_pp_hash` is set to `SHA3-256(merch_PS_pk, merch_addr, merch_pk)`.

### Tezos-related node initialization
We assume the merchant runs or connects to a `tezos-node` that has been initialized correctly and securely. This means that the node has successfully established a connection to the P2P network and connected to a list of bootstrapped and trusted peers. It is assumed that the node runs a version of tezos that includes support for the **Edo** protocol or later.

### Revocation database initialization
The merchant must initialize a database `revocation_DB`. This database is used by both the `zkAbacus` and `TezosEscrowAgent` components, for processing payments and closing escrow accounts, respectively. More information on the merchant's longterm database is provided [here](merch-db.md).
