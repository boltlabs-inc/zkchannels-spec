# Global System Setup
Commitment parameters. These parameters are used for generic commitments in zkAbacus. (XX TODO define as g2 from BLS12-381 and hash to curve of some sensible string for the other generator)

# Setup for new merchants

A party who wishes to act as a zkChannel merchant must create and publish parameters for use across all of their zkChannels: 

1. Blind signature public key pair. The merchant must use this key pair to sign all channel state initializations and updates in zkAbacus. This keypair is also used as a condition in the Tezos smart contract for each channel. (XX: TODO break this into parts bc we will only send the G2 part to Tezos for verification?)
2. Range proof keys and signatures. The zkAbacus.Pay protocol requires a separate set of parameters to prove that updated channel balances are valid.


## Blind signing keys
Generate a [new](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) Pointcheval-Sanders keypair of length 5.
Publish the public key from the pair.

## Commitment keys
Generate [new](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/pedersen_commitments.rs#L41) Pedersen parameters of length 1. Publish the whole set.

## Range proof parameters

A range proof demonstrates that a value (in our case, an updated balance) is non-negative. Specifically, it can prove that an integer is in the range `[0, u^l)`, for some values `u`, `l`. 

TODO: Confirm choice of `u`,`l` (probably 128,9)

Generate a [new](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_keys.rs#L69) Pointcheval-Sanders keypair of length 5.
With this keypair, separately [sign](https://github.com/boltlabs-inc/libzkchannels-crypto/blob/main/libzkchannels-crypto/src/ps_signatures.rs#L64) each value from 0 to `u`.

Publish the public key and the signatures.

## TODO 
Should this page also describe the merchant setting up and publishing a server IP address? Is there other Tezos-related initialization? Are there other keys?
