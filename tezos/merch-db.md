## Merchant Revocation Database
The merchant database `revocation_DB` is used to enforce correct `zkAbacus` payments and `TezosEscrowAgent` account closes.

This databse should consist of two independent tables: one storing a set of (non-null) nonces, and one storing a set of revocation lock/secret pairs (the secret may be NULL but the revocation lock must be present).

### Operations

* For nonces: the only supported operation should take a nonce as input, and execute the atomic sequence "read whether this nonce is present, insert it if not, and return a boolean indicating whether it was already present". Ensuring that this is an atomic transaction means that of any number of concurrent Pay sessions with the same nonce, only one will succeed, while the rest will fail because the nonce will already have been present.

* For revocation locks: the only supported operation should take a revocation lock and optional secret, and execute the atomic sequence "read and store all pairs of lock/optional-secret, insert this lock/optional-secret, and return the set previously stored". This procedure will be called in several situations:

* In `zkAbacus.Pay`, this procedure will be called with the lock and secret the customer reveals, and the Pay session should be aborted if it returns anything other than the empty set.
When the merchant receives notification of a unilateral customer close, this procedure will be called with the lock found in the on-chain transaction and the NULL revocation secret. If the returned set is empty, that means the customer closed on the most recent state of the channel. If it contains at least one revocation secret, this secret will be used to enact a punitive close. If it contains only the tuple (lock, NULL), this means the customer simultaneously performed a mutual close while closing unilaterally, both on the most recent state. This is weird behavior, but not problematic to the integrity of the protocol.

* When the merchant is processing a mutual close, this procedure will be called just like in a unilateral customer close. If the returned set is empty, the mutual close should proceed as normal. If the returned set contains only (lock, NULL), it means there is an on-chain customer close transaction on the most recent channel state at the time of the mutual close session (see above: this is weird behavior but not problematic). If there is a secret in the set, the customer is trying to mutual-close on an old channel state, so the merchant aborts the mutual close. If the merchant knows the identity of the customer, they may wish to take some sort of (possibly punitive) action if they see this behavior.

### Query optimizations
 The single atomic procedure specified for fetching and inserting revocation lock/secret pairs could be specialized if necessary to improve performance: since Pay only cares whether the set is empty or not, we could return a mere boolean instead of a set of tuples for that query. Similarly, we could return only one record for the unilateral and mutual close queries, since any secret will do just as well: return (lock, NULL) if it is the only record with lock, otherwise return the first (lock, secret) pair with a defined secret.