# Farcaster Specifications

Hello voyager and welcome to the Farcaster Specifications! The RFCs describe components that glue together to form a swap protocol between two blockchain based assets, currently focusing on Bitcoin and Monero.

## Overview

Below is an index of the RFCs, loosely in the order we recommend studying them to familiarize yourself with the project.

### [00. Introduction](https://hackmd.io/asVEnqtFR-iL2QIvVxwqjQ)
Introduction, Glossary and Terminology Guide that will help you familiarize with the Farcaster and Atomic swap terminology.

### [01. High Level Overview](https://hackmd.io/hY3vS7YeTLSBQGyNQiNhPA?both)
Describes the high level concepts associated to the protocol such as roles and phases implemented inside Farcaster.

### [02. User Stories](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)
Presents from a user perspective how participants interact during the swap depending on their roles.

### [03. Farcaster Architecture](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA)
Provides an overview of the technical architecture of the Farcaster software stack. Lists and explains the micro-services that compose Farcaster.

### [04. Protocol messages](/M0uYws_5S7K6k1j5l8b6qw)
Specifies the messages exchanged between daemons during a swap and the peer-to-peer management between daemons.

### [05. Tasks & Blockchain Events](/0UBnjLo3QzWx_ReejLHgYQ)
Specifies the tasks that syncers make available to daemons for the monitoring of chain-state through blockchain events and the broadcasting of transactions.

### [06. Instructions & State Digests](/ASzYCe0oQsSbDwXLRlgnQw)
Specifies the instructions that daemon makes available to clients for piloting the swap and state digests send to clients by the daemon.

### [07. Cryptographic Setup](https://hackmd.io/ElXeBEuTQ1Opa0g9yHJszQ)
Specifies the cryptographic primitives used to transfer secrets through transactions with adaptor signatures and specifies the cryptographic setup required at the beginning of a swap to guarentee funds safety.

### [08. Transactions](/YfMko2WPR9iITsw4MsLcPA)
Specifies the Bitcoin and Monero transactions involved in a swap with their temporal safety. Variants of Bitcoin transactions are described depending on the SegWit version and multi-signature protocol used.

## Acknowledgments

This project was funded by the Monero community through the [Community Crowdfunding System (CCS)](https://ccs.getmonero.org/). Thanks to all contributors and generous anonymous donnors!