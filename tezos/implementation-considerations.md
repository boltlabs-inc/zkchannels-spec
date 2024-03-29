# Implementation Considerations
* [Customer Privacy](#customer-privacy)
* [Security Comments](#security-comments)

# Customer Privacy
- _The merchant must use the same zkAbacus public parameters across all channels_. Tne size of a customer's anonymity set is determined by how many channels are open with a given merchant using a particular set of public parameters, namedly those used in zkAbacus, which consist of a zkAbacus public key and range proof parameters. The customer must therefore have a mechanism to retrieve these parameters that provides the assurance not only that the parameters are indeed from the merchant, but also that the merchant has only set of parameters for all their customers. Potential options for this include the provision and use of an on-chain registry service for merchant parameters.

- _Traffic analysis_. The zkChannels protocol does not provide any protection against a network-level adversary. Implementations should use a privacy-enhancing networking tool such as Tor.

- _Timing attacks on closes after adversarial merchant aborts_. The zkChannels protocol provides _cryptographic_ unlinkability if a merchant aborts during a customer payment. That is, the customer is always able to close using information that the merchant cannot, from the content of the close, use to link a channel to a particular aborted payment. However, this does not prevent _timing attacks_. Implementations should therefore consider mechanisms to avoid trivial trivial timing attacks in this case, such as by including a random delay interval before closing a channel after an aborted payment.

# Security Comments

- _Optimizing merchant storage_. A merchant may periodically drop closed channels from memory, but must not naively drop any information from their revocations database. In order to re-initialize this database, the merchant would have to stop processing payments and close all their channels at the same time.

- _Self delay values_. The parties must agree on the values used for on-chain transaction timeouts. If global defaults are used across all of a merchant's channels, these should be set to relatively large values, thereby allowing all parties sufficient time to respond even in the presence of periodic network failures. If parties wish to specify channel-specific delay values, these must be set in accordance with each party's risk tolerances: the customer should choose the self-delay period of `expiry` and the merchant should choose the self-delay period of `custClose`.