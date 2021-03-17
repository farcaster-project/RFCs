<pre>
  State: draft
  Created: 2020-11-11
</pre>

# 03. Farcaster Architecture

## Overview

This RFC describes the overall architecture and how the software stack of Farcaster is conceptually organized. It includes a role definition of the three main components: `client`, `daemon`, and `syncers`, and a high-level overview of their interactions.

## Table of Contents

  * [Global Architecture](#global-architecture)
    * [Main components](#main-components)
    * [Components interaction](#components-interaction)
    * [Networking stack](#networking-stack)
    * [Client & Daemon segregation rationale](#client--daemon-segregation-rationale)
  * [The `client` component](#the-client-component)
    * [Client-Daemon communication](#client-daemon-communication)
    * [Why two bidirectional message types?](#why-two-bidirectional-message-types)
  * [The `daemon` component](#the-daemon-component)
    * [Counter-party daemon communication](#counter-party-daemon-communication)
    * [Loopback: self-generated input messages](#loopback-self-generated-input-messages)
  * [The `syncer` component](#the-syncer-component)
    * [Blockchain communication](#blockchain-communication)

## Global Architecture

This architecture aims to provide flexibility and ease evolution of components or protocol for the future. Different use cases require different type of IT infrastructure and different threat models. Splitting the stack into three main components provide this flexibility and allow components to evolve solely by keeping the correct interfaces defined in these RFCs.

### Main components

We segregated three main conceptual components: `client`, `daemon`, and `syncers`.

- A `client` drives a swap and is the interface for a user to interact with the system,
- The `daemon` orchestrates the swap protocol execution, and
- A `syncer` maintains the protocol state and a specified blockchain state in sync.

The figure below represents the general architecture based on these three main types of components.

![Farcaster High-Level Components Architecture](./03-farcaster-architecture/global-farcaster-architecture.png)
*Fig 1. Farcaster High-Level Components Architecture*

Each component have a specific assigned role, they all work together to complete a swap. The following table summarizes different aspects of each component.

|                              | `client`                                                          | `daemon`                                                     | `syncer`                                        |
|------------------------------|-------------------------------------------------------------------|--------------------------------------------------------------|-------------------------------------------------|
| Definition                   | a program that controls the daemon and displays the current state | a program that executes the core protocol in a state machine | a program that talks with a specific blockchain |
| Cryptographic keys & secrets | private & public                                                  | public only`*`                                               | public only`*`                                  |
| Client/User                  | end-user                                                          | `client`, counter-party `daemon`                             | `daemon`                                        |
| Availability                 | present at the start and to sign                                  | mostly online, channel of communication between parties      | always online                                   |
| Communicates with            | `daemon`                                                          | `client`, `syncer`, counter-party `daemon`                   | `daemon`, blockchain                            |
| Transactions                 | creates all transactions, signs                                   | verifies transactions and signatures                         | listens for and publishes transactions          |
| Protocol state               | doesn't understand protocol, but can represent its state          | understands the protocol, but can't sign                     | doesn't understand protocol                     |

 `*` the exception is any keys needed to read the blockchain state and detect transactions or amounts, e.g. a private view key for the Monero accordant blockchain.

### Components interaction

Each swap component is represented as a black box that consumes input messages and produces output messages. Each input and output message is a typed message. Components subscribe to types of messages, e.g. the client may not consume messages produced by syncers but will subscribe to daemon's messages.

Typed messages are defined to specified the interfaces between the components and to document what type of data is needed to be exchanged between this three high level components. In reallity the daemon can be split into multiple small services, but the interaction with the `client` and the `syncers` must follow the defined interfaces.

![Typed messages exchanged between components](./03-farcaster-architecture/messages-architecture.png)
*Fig 2. Typed messages exchanged between components*

It is worth noting that this diagram (Fig. 2) only shows one syncer, but a syncer per blockchain is required. Conceptually even more than one syncer per blockchain makes sense if you don't run or trust the syncer you are using, in that case, one can aggregate and compare different data sources and detect discrepancies.

### Networking stack

Typed messages are strictly defined with their serialization and the inter-daemon networking stack is constrained and defined in [04. Protocol Messages](./04-protocol-messages.md), but we don't restrict the networking stack between `daemon`, `client` and `syncer`s. The technological choice is left up to the implementation, but must support running all interactions over the network, whenever required by the user.

### Client & Daemon segregation rationale

The rationale behind segregating the client and the daemon is currently not motivated by security reasons - the client creates and signs the transactions blindly based on messages received from the daemon, therefore implying full trust (see *Security considerations* in [06. Datum & Instructions](./06-datum-and-instructions.md#security-considerations)).

The client is the only component that has access to secret keys that guarantee the safety of the swapped funds.

The aim of this segregation is to improve flexibility and extensibility added by making the client peripheral to the fundamental swap stack - that is, other clients might be created, such as:

- Clients supporting hardware wallets
- Mobile applications (that may run the daemon in the background or in a private server),
- Heavy- or light-weight desktop GUIs,
- Scripted/automated backend clients (e.g. run by market makers, OTCs etc)

We give a partial example of a `cli` client and a simple example of a `gui` client in [02. User Stories](./02-user-stories.md) to illustrate two implementation of a client.

## The `client` component

The client is the only component aware of the user's private keys and acts as a "swap wallet", creating and signing the blockchain transactions and communicating with the daemon through typed messages.

### Client-Daemon communication

Client and daemon communicate via `datum` and `instruction` messages (see [06. Datum & Instructions](./06-datum-and-instructions.md)). The architecture must allow a client and a daemon to run on different machines.

`datum` and `instruction` messages are asynchronous and may fail, the daemon must handle missing messages from a client, and vice versa, and may modify the swap state in response.

Client and daemon must have an initialization protocol allowing one or the other to recover from a past swap state (see [09. Swap State](./09-swap-state.md)).

Client passes `datum` and `instruction` messages to the daemon. That is, client may at the user's discretion fire one of the possible protocol transitions at each moment in time. A subset of transitions may require transactions, signatures, or keys as input which are transmitted via `datum` messages. Upon receiving a client's message, daemon must undertake the actions associated with the fired transition, and update the swap state accordingly. Daemon updates client with the same type of messages: `datum` and `instruction`, allowing the client to update its local state and produce the next `datum` messages needed by the daemon.

### Why two bidirectional message types?

We distinguish between `datum` messages who carry essential data for performing the swap such as keys, signatures, transactions, parameters, etc. and `instruction`s that control the flow of the swap and may come from the user or the counter-party daemon, such as an `abort` operation.

## The `daemon` component

The Daemon is the central component responsible for orchestrating the protocol execution.

Its main function is to manage the safe progression of the execution of the cross-chain atomic swap in response to the intrinsic and extrinsic messages it receives - respectively coming from itself and from the components it directly interacts with.

The daemon must be fully aware of the complete state of the cross-chain atomic swap execution. To achieve that it must listen to:

- Counter-party `daemon` protocol messages via inter-daemon communication (see [04. Protocol Messages](./04-protocol-messages.md)),
- Blockchain events from both blockchains via `syncer`s (see [05. Tasks & Blockchain Events](./05-tasks-and-events.md)),
- User's data and instructions via a `client` (see [06. Datum & Instructions](./06-datum-and-instructions.md)), and
- Self-produced loopback messages

The Daemon must create a constrained runtime environment for executing the protocol, that only permits valid protocol transitions at all times. To achieve this, a Petri net model of the protocol may be used to constrain the runtime environment that executes the user's respective swap role in the protocol (see [01. High Level Overview](./01-high-level-overview.md)), by only authorizing firing valid enabled protocol transitions.

### Counter-party daemon communication

The Daemon has a bidirectional communication channel with the swap counter-party's daemon. This inter-daemon communication is used to pass protocol messages between the swap counterparties. To bootstrap this communication channel we distinguish two phases: the negotiation and the swap phase. The negotiation phase allows users to discover swaps through public offers and connect their daemons (see [02. User Stories](./02-user-stories.md)).

### Loopback: self-generated input messages

The daemon may generate self-addressed messages. Those messages may be used to trigger transitions only based on the daemon's state, such as timers. Those transitions can e.g. represent the absence of counter-party daemon communication during a period of time, which may trigger the swap cancellation process.

## The `syncer` component

A syncer is specific to a blockchain and can handle a list of `tasks` related to it. Those `tasks` will be completed in different manners depending on the blockchain type and/or the blockchain state. Syncers allow the daemon to abstract a part of the logic needed to interact with a blockchain with a defined interface composed of `tasks` and `blockchain events` (see [05. Tasks & Blockchain Events](./05-tasks-and-events.md)).

### Blockchain communication

A syncer can choose how to interact with its specified blockchain, this can be done solely through RPC calls or involve more advanced features such as e.g. consuming the 0MQ streams proposed by a full node.
