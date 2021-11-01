# Implementation Considerations
* Customer Privacy [#Customer-Privacy]

# Customer Privacy
- _The merchant must use the same zkAbacus public parameters across all channels_. Tne size of a customer's anonymity set is determined by how many channels are open with a given merchant using a particular set of public parameters, namedly those used in zkAbacus, which consist of a zkAbacus public key and range proof parameters. The customer must therefore have a mechanism to retrieve these parameters that provides the assurance not only that the parameters are indeed from the merchant, but also that the merchant has only set of parameters for all their customers. Potential options for this include the provision and use of an on-chain registry service for merchant parameters.

- 