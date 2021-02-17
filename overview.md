# Overview

Welcome to the Farcaster Specifications! The RFCs describe components that glue together to form a swap protocol between Bitcoin and Monero.

Below is an index of the RFCs, loosely in the order we recommend studying them to familiarize yourself with the project.

## [00. Introduction]()
Glossary and Terminology Guide.

## [01. User Stories / High Level Protocol](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)
High-level overview of roles and phases of the protocol. Good starting point with this RFC to understand the roles participants play from a high level. This RFC links to other RFCs where a deeper dive into the details may be desired. Cross-reading with [Farcaster Architecture](/vTCjO2-ySr6SB7ObuJMhnA) is recommended.

## [02. Farcaster Architecture](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA)
Provides an overview of the technical architecture of the Farcaster software stack. Lists and explains the micro-services that compose Farcaster.

## [03. Protocol messages](/M0uYws_5S7K6k1j5l8b6qw)
Specifies the messages exchanged between daemons during a swap and the peer-to-peer management between daemons.

## [04. Tasks & Blockchain Events](/0UBnjLo3QzWx_ReejLHgYQ)
Specifies the tasks that syncers make available to daemons for the monitoring of chain-state through blockchain events and the broadcasting of transactions.

## [05. Instructions & State Digests](/ASzYCe0oQsSbDwXLRlgnQw)
Specifies the instructions that daemon makes available to clients for piloting the swap and state digests send to clients by the daemon.

## [06. Transactions](/YfMko2WPR9iITsw4MsLcPA)
Specifies the Bitcoin and Monero transactions involved in a swap with their temporal safety. Variants of Bitcoin transactions are described depending on the SegWit version and multi-signature protocol used.

## [07. Cryptographic Setup](https://hackmd.io/ElXeBEuTQ1Opa0g9yHJszQ)
Specifies the cryptographic primitives used to transfer secrets through transactions with adaptor signatures and specifies the cryptographic setup required at the beginning of a swap to guarentee funds safety.