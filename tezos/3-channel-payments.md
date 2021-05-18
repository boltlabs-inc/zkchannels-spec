# Channel Payments
Say a customer `C` wishes to purchase a good or service, a description of which we denote by `x`,
from the merchant `M` using their previously established zkChannel with identifier `cid`. 
Assume the cost is `E`.
1. To make a payment on channel `cid`, the customer `C` establishes a session with the merchant
`M` and sends a _payment request message_ containing the tuple (`x`, `E`) to `M`.
2. The merchant checks (`x`, `E`) and decides whether to accept or reject the payment. If the former, they send `accept`; if the latter, they send `reject` and abort.
3. Upon receipt of `reject`, the customer `C` aborts the session. Upon receipt of `accept`, the customer and merchant run 

        zkAbacuz.Pay((pk'_M, E, (s_i, pt_i)_C, (S_1, S_2, sk'_M)_M))

where `s_i` and `pt_i` are the most recent channel state and associated payment tag for channel `cid`, respectively.

4. Whatever the purpose of payment, the counter `ctr_M` can be used by the merchant to make decisions about whether a payment is complete and/or a service should be provided. That is, the merchant uses `ctr_M` and the context of the payment `x` to determine further action, such as the provision of a requested good or service. More precisely, the merchant extracts the resulting `ctr_M` from `zkAbacuz.Pay()` output and behaves as follows: 

    * If `ctr_M` = `⊥`, the payment has unequivocably failed. No good or service should be provided.

    * If `ctr_M` = `revocation-incomplete`, then if `E` < 0, the merchant should consider the payment complete. Otherwise, the merchant should consider the payment incomplete and no good or service should be provided.
	* If `ctr_M` = `revocation-complete`, then the merchant should consider the payment complete.


5. The customer extracts the resulting `ctr_C`$ from `zkAbacuz.Pay()` output
and does the following:
    * If `ctr_C` = `revocation-incomplete`, the customer sets `status` = `frozen`, initiates unilateral closure on state `s_bar_i` as specified in (XX: add reference).
    * If `ctr_C` = `revocation-complete`, the customer sets `status` = `frozen`, initiates unilateral closure on state $`s_bar_i+1` as specified in (XX: add reference).
    * If `ctr_C` = `state-updated`, then the customer should expect to receive the requested good or service, and may continue to make additional payments on the channel.