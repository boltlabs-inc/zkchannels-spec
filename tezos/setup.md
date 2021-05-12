# Global System Setup
## Elliptic Curve Selection.
We use an implementation of the pairing-friendly curve [BLS12-381](https://crates.io/crates/bls12_381) with security level approximately 128 bits.

## Commitment parameters. 
We use Pedersen commitments on messages of length one in the pairing group `G1` of BLS12-281. We set parameters as the pair `(g1, h)`, where `g1` is the standard generator defined in the BLS12-381 crate and `h` is a hash to curve of string `'zkChannels Pedersen generator'`.

## Hash function.
We use SHA3-256.

## Pointcheval-Sanders signatures public parameters.
We use the pairing provided by the BLS12-381 crate.

# Merchant Setup

A party who wishes to act as a zkChannel merchant must create and publish the following parameters for use across all of their zkChannels: 

* Blind signature public key pair. The merchant must use this key pair to sign all channel state initializations and updates in zkAbacus. This keypair is also used as a condition in the Tezos smart contract for each channel. (XX: TODO break this into parts bc we will only send the G2 part to Tezos for verification?)
* Range proof keys and signatures. The zkAbacus.Pay protocol requires a separate set of parameters to prove that updated channel balances are valid.

## Blind signing key

Generate a [new Pointcheval-Sanders keypair](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) for signatures on message tuples of length `5`.
Publish the public key from this pair.

## Range proof parameters

A range proof demonstrates that a value (in our case, an updated channel balance) is non-negative. Specifically, we use range proofs to show that an integer is in the range `[0, u^l)`, for integer values `u`, `l`. We set `u = 128` and `l = 9`.

Generate a [new Pointcheval-Sanders keypair](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) for signatures on message tuples of length `1`.
With this keypair, separately [sign](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_signatures.rs#L64) each integer in the range `[0, u-1]`.

Publish the resulting public key and signatures.

## TODO 
Should this page also describe the merchant setting up and publishing a server IP address? Is there other Tezos-related initialization? Are there other keys?
