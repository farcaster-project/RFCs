<pre>
  State: draft
  Created: 2020-11-11
</pre>

# Farcaster Architecture

## Overview

This RFC describe the overall architecture and how the software stack of Farcaster is organized. Description of the swap state organization and its recovery process and an high level overview of the three main components: client, daemon, and syncers.

## Table of Contents

  * [Global Architecture](#global-architecture)
    * [Client & Daemon segregation rationale](#client--daemon-segregation-rationale)
    * [Components interaction](#components-interaction)
  * [Daemon](#daemon)
    * [Counter-party daemon communication](#counter-party-daemon-communication)
    * [Blockchain communication](#blockchain-communication)
    * [Client-Daemon communication](#client-daemon-communication)
    * [Loopback: self-generated input messages](#loopback-self-generated-input-messages)
  * [Syncer](#syncer)
  * [Client](#client)
  * [Swap State](#swap-state)
    * [Recover state](#recover-state)
    * [Update state](#update-state)
    * [Recovery process between components](#recovery-process-between-components)

## Global Architecture

We segregated three main conceptual components: `client`, `daemon`, and `syncers`.

- The `client` drives the swap,
- the `daemon` orchestrates the swap protocol execution and
- the `syncer` maintains the protocol state and the blockchain state in sync

The figure below represents the general architecture based on these three main type of components.

![Farcaster High Level Components Architecture](https://github.com/farcaster-project/RFCs/raw/master/images/arch.png)
*Fig 1. Farcaster High Level Components Architecture*

The following table summarizes different aspects of each component.

|                 | `client`                    | `daemon`                                        | `syncer`     |
|-----------------|----------------------------------|------------------------------------------------------|--------------------|
| Definition   | a program that controls the daemon and displays the current state | a program that executes the core protocol in a state machine | a program that talks with a specific blockchain |
| Cryptographic keys & secrets | private & public    | public only`*`                                        | public only`*`      |
| Client/User  | end-user                         | `client`, counter-party `daemon`                | `daemon`        |
| Availability          | present at the start and to sign | mostly online, channel of communication between parties | always online      |
| Communicates with | `daemon`                      | `client`, `syncer`, counter-party `daemon` | `daemon`, blockchain |
| Transactions    | signs                            | creates all transactions, verifies signatures        | listens for and publishes transactions  |
| Protocol state  | doesn't understand protocol, but can represent its state              | understands the protocol, but can't sign             | doesn't understand protocol |

 `*` the exception is any keys needed to read the blockchain state and detect transactions or amounts, e.g. a private view key for the Monero accordant blockchain.

### Client & Daemon segregation rationale

The rationale behind segregating the client and the daemon currently is not for security reasons - the client signs the transactions received from the daemon blindly, implying full trust (see *Security considerations* in [06. Instructions & State Digests](./06-instructions-and-digests.md#security-considerations) RFC).

The client is the only component that has access to secret keys.

The aim of this segregation is to improve flexibility and extensibility added by making the client peripheral to the swap stack, that is, other clients might be created, such as:

- clients supporting hardware wallets
- mobile applications (that may run the daemon in background or in a private server), 
- heavy- or light-weight desktop GUIs, 
- scripted/automated backend clients (e.g. run by market maker, OTCs etc)

### Components interaction

Each swap components is represented as a black box that consumes input messages and produces output messages. Each input and output message is a typed message. Components subscribe to type of messages, e.g. the client may not consume messages produced by syncers but will subscribe to daemon's messages.

![Typed messages exchanged between components](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/arch-global.png)
*Fig 2. Typed messages exchanged between components*

It is worth noting that this diagram (Fig. 2) only show one syncer, but a syncer per blockchain is required. Conceptually even more than one syncer per blockchain make sense if you don't run or trust the syncer you are using, in that case one can aggregate and compare different data sources and detect discrepancies.

## Daemon
The Daemon is the central component responsible for orchestrating the protocol execution. 

Its main function is to manage the safe progression of the execution of the cross-chain atomic swap in response to the intrinsic and extrinsic messages it receives - respectively coming from itself and from the components it directly interacts with. 

The Daemon MUST be fully aware of the complete state of the cross-chain atomic swap execution. To achieve that it must listen to:

- counter-party daemon protocol messages via inter-daemon communication (see [04. Protocol Messages](./04-protocol-messages.md)), 
- blockchain events from both chains via syncers (see [05. Tasks & Blockchain Events](./05-tasks-and-events.md)),
- user's instructions via Client communication (see [06. Instructions & State Digests](./06-instructions-and-digests.md)), and
- self-produced loopback messages

The Daemon must create a constrained runtime environment for executing the protocol, that only permits valid protocol transitions at all times. To achieve that a petrinet model of the protocol may be used to constrain the runtime environment that executes the user's respective swap role in the protocol, by only authorizing firing valid enabled protocol transitions.

### Counter-party daemon communication
The Daemon has a bidirectional communication channel with the swap counter-party's daemon. This inter-daemon communication is used to pass protocol messages between the swap counterparties (see [04. Protocol Messages](./04-protocol-messages.md)).

### Blockchain communication
A Daemon does not interact with a blockchain fullnode or a public API directly. Actions and queries are done via a syncer through `tasks` and `blockchain events` (see [05. Tasks & Blockchain Events](./05-tasks-and-events.md)).

A syncer handles `tasks` requests from a daemon and produces `blockchain events` according to blockchain state changes.

### Client-Daemon communication
Client and daemon communicate via `Instruction` and `State Digest` messages (see [06. Instructions & State Digests](./06-instructions-and-digests.md)). The architecture must allow a client and a daemon to run on different machines.

### Loopback: self-generated input messages
The daemon may generate self-addressed messages. Those messages may be used to trigger transitions only based on daemon's state, such as timers. Those transitions can e.g. represent the absence of counter-party daemon communication during a period of time, which may trigger the swap cancellation.

## Syncer
A syncer is specific to a blockchain and can handle a list of `tasks` related to it. Those `tasks` will be completed in different manners depending on the blockchain type and/or the blockchain state. Syncers allow the daemon to abstract a part of the logic needed to interact with a blockchain with a define interface composed of `tasks` and `blockchain events` (see [05. Tasks & Blockchain Events](./05-tasks-and-events.md)).

## Client
The client is the only component aware of the user's private keys and acts as a "swap wallet", signing the blockchain transactions and piloting the swap through `instructions`.

Instruction messages are asynchronous and may fail, the daemon must handle missing instructions from a client and may modify the swap state in response.

Client and daemon must have an initialization protocol allowing one or the other to recover from a past swap state.

Daemon updates client with `state digest` messages to move forward with the swap protocol execution. `state digest` contain a summary of the current swap state so the client can update his view on the swap state.

Client passes `instructions` to the daemon. That is, client may at the user's discretion fire one of the possible protocol transitions at each moment in time. A subset of transitions may require signed protocol messages and/or signed blockchain transactions as input. After client's instruction, daemon must undertake the actions associated with the fired transition, and update the swap state accordingly.

## Swap State
The swap state encodes the step of the protocol execution the user is currently in, and it is handled by the daemon. For each given state, zero or more transitions are enabled as a function of a list of valid inputs. Inputs can be: (i) blockchain events, (ii) protocol messages, (iii) client instructions, or (iv) daemon loopback messages. When a swap state receives an input, a transition to a new state may produce outputs. Outputs can be: (i) syncer tasks, (ii) protocol messages, (iii) client state digest messages, or (iv) daemon loopback messages.

Valid transitions from one state to other states are described by the protocol. Upon swap state update, daemon must update the client through state digest messages.

For each swap state, only a subset of inputs is valid. The daemon must contextually filter all of its inputs before applying them. Filtering can happen in parallel for every stream of inputs. All filtered streams are then combined, each element of this final stream are applied to the current state.

The first task of the combined input stream must be a save on disk. After each transition the swap state must be saved on disk as the last checkpoint. The daemon must be able to start with a given checkpoint, a saved stream of inputs and a protocol.

The swap state can be viewed as an ordered set of inputs. On a crash the daemon must be able to load the last checkpoint and apply the stream of inputs (saved and incoming inputs) uncontained in the loaded checkpoint.

By ensuring that daemon output messages can be replayed safely, like tasks, the work already performed since the last saved checkpoint can be replayed safely.

### Recover state
`L` is the complete set of valid logs produced by valid protocol executions and `Σ` is the set of protocol states.

There is a function, `recover_state` or `r`,

```
                L → Σ
                r(logs) = σ
```

that takes the time-ordered logs composed of 

- blockchain event 1, `Ev1`
- blockchain event 2, `Ev2`
- client instructions, `Inst`
- counter-party protocol messages, `Prot2`
- self protocol messages, `Prot1`
- internal-timers, `Loopback`
- published task, `Task1`
- published task, `Task2`

and that outputs a protocol state, `σ`.

### Update state
There is a function, `update_state` or `u`,

```
                Σ × L → Σ
                u(σt−1, Δlogs) = σt
```

where `σt−1` is the previous state and `σt` is the final state, and `Δlogs` are the logs between `t-1` and `t`.

Background on [petrinets](http://petrinet.org/) for the following.

The protocol state may be encoded as markings of a petrinet.

In practice the code may be implemented as a fold operation over a stream of events taking a petrinet marking as initial state that internally outputs the new marking (which encodes the new protocol state). Then tap the wire after the fold operation to monitor the state.

### Recovery process between components
The recovery process is different depending the components. Client and daemon are trusted, the prerequisits are not the same as e.g. inter-daemon.

#### Inter-daemon
Daemon to daemon communication must have a mechanism to reconnect and gracefully recover from the latest saved state. This is the most critical recovery of the protocol as recovering to the wrong state may permit counter-party to steal funds.

TODO(h4sh3d): elaborate a be more precise

#### Client-Daemon
Both daemon and client must implement mechanisms to restablish their connection and safely recover from a previously saved state.

TODO(h4sh3d): elaborate a be more precise

#### Syncer-Daemon
As tasks can be replayed safely the daemon and syncer does not need any particular recovery mechanism.