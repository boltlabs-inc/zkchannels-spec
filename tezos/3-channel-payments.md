# Channel Payments

A customer wishes to purchase a good or service from the merchant using their zkChannel with channel identifier `cid`. We denote the good or service by `x` in the description below and the cost is an amount `amt`.

1. The customer sends a _payment request message_ containing the tuple `(x, amt)` to the merchant.
2. The merchant checks the payment request message and decides whether to accept or reject. If the former, they send `accept`; if the latter, they send `reject` and abort.
3. Upon receipt of `reject`, the customer aborts the session. Upon receipt of `accept`, the customer and merchant run `zkAbacus.Pay()` for amount `amt`. 
The customer must have status `Ready` and must use the current channel state data to process the payment. 
The merchant must have read and write access to their [revocation database](1-setup.md#Revocation-database-initialization) `revocation_DB` and their [blind signing key](1-setup.md#Blind-signing-key-generation).

    During execution of `zkAbacus.Pay()`, the customer will transition their status from `Ready` to `Started` to `Locked` and finally to `Ready`. For more details on this functionality, see Section 3.3.2 of the 
[zkChannels Private Payments Protocol](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf).

4. The merchant receives a success indicator from `zkAbacus.Pay()`, which can be used to decide whether or not the requested service should be provided:

    * If the merchant receives a failure indicator, no good or service should be provided.

    * If the merchant receives `revocation-incomplete`, then if `amt` represents a negative value (i.e., the payment request is for a refund), the merchant should consider the payment complete. This is because, in this case, the customer has a way to close on their updated state, in which the refund has been successfully applied. If `amt` represents a positive value, the merchant should consider the payment incomplete and no service should be provided.

	* If the merchant receives `revocation-complete`, then the merchant should consider the payment complete. In this case, the customer can no longer close on the previous state. In this case, the merchant should provide the requested service.


5. The customer also receives a success indicator from `zkAbacus.Pay()` and does the following:
    * If the customer receives `revocation-incomplete`, they will have a channel status of `Started`. They initiate a [unilateral customer close](4-channel-closure.md#unilateral-customer-close) on their current state, not including the attempted payment.
    * If the customer receives `revocation-complete`, they  will have a channel status of `Locked`. They initiate a [unilateral customer close](4-channel-closure.md#unilateral-customer-close) on the updated state, the balances of which reflect a payment of amount `amt`.
    * If the customer receives `state-updated`, the channel status is `Ready`. They should expect to receive the requested good or service and may continue to make additional payments on the channel.
