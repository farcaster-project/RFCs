# Farcaster Specifications

Hello hitchhiker and welcome onboard to the Farcaster Specifications! The RFCs describe components that glue together to form a swap protocol between two blockchain based assets, currently focusing on Bitcoin and Monero.

## Overview

Below is an index of the RFCs, loosely in the order we recommend studying them to familiarize yourself with the project.

### [00. Introduction](./00-introduction.md)
Introduction, Glossary and Terminology Guide that will help you familiarize with the Farcaster and Atomic swap terminology.

### [01. High Level Overview](./01-high-level-overview.md)
Describes the high level concepts associated to the protocol such as roles and phases implemented inside Farcaster.

### [02. User Stories](./02-user-stories.md)
Presents from a user perspective how participants interact during the swap depending on their roles.

### [03. Farcaster Architecture](./03-farcaster-architecture.md)
Provides an overview of the technical architecture of the Farcaster software stack. Lists and explains the micro-services that compose Farcaster.

### [04. Protocol Messages](./04-protocol-messages.md)
Specifies the messages exchanged between daemons during a swap and the peer-to-peer management between daemons.

### [05. Tasks & Blockchain Events](./05-tasks-and-events.md)
Specifies the tasks that syncers make available to daemons for the monitoring of chain-state through blockchain events and the broadcasting of transactions.

### [06. Instructions & State Digests](./06-instructions-and-digests.md)
Specifies the instructions that daemon makes available to clients for piloting the swap and state digests send to clients by the daemon.

### [07. Cryptographic Setup](./07-cryptographic-setup.md)
Specifies the cryptographic primitives used to transfer secrets through transactions with adaptor signatures and specifies the cryptographic setup required at the beginning of a swap to guarentee funds safety.

### [08. Transactions](./08-transactions.md)
Specifies the Bitcoin and Monero transactions involved in a swap with their temporal safety. Variants of Bitcoin transactions are described depending on the SegWit version and multi-signature protocol used.

## Acknowledgments

This project was funded by the Monero community through the [Community Crowdfunding System (CCS)](https://ccs.getmonero.org/). Thanks to all contributors and generous anonymous donnors!
