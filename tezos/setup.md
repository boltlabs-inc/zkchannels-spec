# Global System Setup

## Commitment scheme parameters
* **Pedersen commitments**: We use Pedersen commitments on messages of length one in the pairing group `G1` of BLS12-381. We set parameters as the pair `(g1, h)`, where `g1` is the standard generator defined in the BLS12-381 crate and `h` is a hash-to-curve of the string `'zkChannels Pedersen generator'`.
* **Hash commitment parameters**:
We use SHA3-256 hashes to instantiate our hash-based commitments.

## Signature schemes
* Pointcheval-Sanders signatures public parameters: We use the pairing defined by the [BLS12-381](https://crates.io/crates/bls12_381) crate. 
* EdDSA signature public parameters: We rely on the public parameters selected by the `tezos-client` for Ed25519.


# Merchant Setup

A party who wishes to act as a zkChannel merchant must create and publish the following parameters for use across all of their zkChannels: 

* **Blind signature public key pair.** The merchant must use this key pair to sign all channel state initializations and updates in `zkAbacus`. This keypair is also used as a condition in the Tezos smart contract for each channel. 
* **Range proof public key and signatures.** The `zkAbacus.Pay` protocol requires a separate set of parameters to prove that updated channel balances are valid. A range proof demonstrates that a value (in our case, an updated channel balance) is non-negative. Specifically, we use range proofs to show that an integer is in the range `[0, u^l)`, for integer values `u`, `l`. We set `u = 128` and `l = 9`. This primitive relies on Poincheval Sanders signatures.

The merchant completes setup as follows.
## Blind signing key

Generate a [new Pointcheval-Sanders keypair](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) for signatures on message tuples of length five.
Publish the public key from this pair.

## Range proof parameters

Generate a [new Pointcheval-Sanders keypair](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) for signatures on message tuples of length one.
With this keypair, separately [sign](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_signatures.rs#L64) each integer in the range `[0, u-1]`.

The merchant publishes the resulting public key and signatures in a config file. The merchant then advertises its server IP address and port for customers to open channels using the `<pub-key>@<ip>:<port>` format.

**Tezos-related Node Initialization.** We assume the merchant runs or connects to a `tezos-node` that has been initialized correctly and securely. This means that the node has successfully established a connection to the P2P network and connected to a list of bootstrapped and trusted peers. It is assumed that the node runs a version of tezos that includes support for the **Edo** protocol or later.