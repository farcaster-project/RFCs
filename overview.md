# Overview
Welcome to the Farcaster RFCs! Here we describe components that glue together to form a swap protocol between Bitcoin and Monero.

Below is an index of our RFCs, loosely in the order we recommend studying them to familiarize yourself with the project.

## [User Stories / High Level Protocol](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)
To get a high-level overview of the protocol, we suggest starting with this RFC to understand the roles participants play from a high level. This RFC links to other RFCs where a deeper dive into the details may be desired. Cross-reading with [Farcaster Architecture](/vTCjO2-ySr6SB7ObuJMhnA) is recommended.  

## [Farcaster Architecture](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA)
This provides an overview of the technical architecture of the Farcaster software stack.

## [Inter-daemon protocol messages](/M0uYws_5S7K6k1j5l8b6qw)
Specifies the messages exchanged between daemons during a swap

## [Transactions](/YfMko2WPR9iITsw4MsLcPA)
Specifies the Bitcoin and Monero transactions involved in a swap. These transactions require activation of Taproot.

## [Bitcoin transactions temporal safety](/Gm-hicyeTpeTM6NoTrSM0Q)
Specifies temporal safety bounds that swap participants must follow to prevent loss of funds

## [Tasks](/0UBnjLo3QzWx_ReejLHgYQ)
Specifies the tasks that syncers make available to daemons for the monitoring of chain-state and the (potentially conditional) broadcasting of transactions

