# Channel Payments

All payments are initiated by the customer. 

Suppose a customer wishes to purchase a good or service from the merchant using their zkChannel with channel identifier `channel_id`. We denote the good or service by `x` in the description below and the cost is an amount `amt`. 

## Customer Requirements 
- The customer's status for the channel with identifier `channel_id` must be `Ready`. 
- The implementation should ensure that the channel has sufficient balance for the payment in question before establishing a session with the merchant (i.e., before sending the first message to the merchant).


## Payment Session
1. The customer sends a _payment request message_ containing the tuple `(x, amt)` to the merchant. 
2. The merchant checks the payment request message and dechannel_ides whether to accept or reject. If the former, they send `accept`; if the latter, they send `reject` and abort.
3. Upon receipt of `reject`, the customer aborts the session. Upon receipt of `accept`, the customer and merchant run `zkAbacus.Pay()` for amount `amt` on the channel with identifier `channel_id`. 
The customer must have status `Ready` and must use the current channel state data to process the payment. 
The merchant must have read and write access to their [revocation database](1-setup.md#Revocation-database-initialization) `revocation_DB` and their [blind signing key](1-setup.md#Blind-signing-key-generation).

    During execution of `zkAbacus.Pay()`, the customer transitions their status from `Ready` to `Started` to `Locked` and finally to `Ready`. For more details on this functionality, see Section 3.3.2 of the 
[zkChannels Private Payments Protocol](https://github.com/boltlabs-inc/blindsigs-protocol/releases/download/ecc-review/zkchannels-protocol-spec-v3.pdf).

4. The merchant receives a success indicator from `zkAbacus.Pay()`, which can be used to dechannel_ide whether or not the requested service should be provided:

    * If the merchant receives a failure indicator, no good or service should be provided.

    * If the merchant receives `revocation-incomplete`, then if `amt` represents a negative value (i.e., the payment request is for a refund), the merchant should consider the payment complete. This is because, in this case, the customer has a way to close on their updated state, in which the refund has been successfully applied. If `amt` represents a positive value, the merchant should consider the payment incomplete and no service should be provided.

	* If the merchant receives `revocation-complete`, then the merchant should consider the payment complete. In this case, the customer can no longer close on the previous state. In this case, the merchant should provide the requested service.


5. The customer also receives a success indicator from `zkAbacus.Pay()` and does the following:
    * If the customer receives `revocation-incomplete`, the channel with identifier `channel_id` has a channel status of `Started`. The customer should initiate a [unilateral customer close](4-channel-closure.md#unilateral-customer-close) on their current state, i.e., with balances that do not reflect the attempted payment.
    * If the customer receives `revocation-complete`, the channel with identifier `channel_id` has a status of `Locked`. The customer should initiate a [unilateral customer close](4-channel-closure.md#unilateral-customer-close) on the updated state, the balances of which reflect a payment of amount `amt`.
    * If the customer receives `state-updated`, the channel with identifier `channel_id` has a status of `Ready`. They should expect to receive the requested good or service and may continue to make additional payments on the channel.
