# Farcaster Architecture

[![hackmd-github-sync-badge](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA/badge)](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA)

<pre>
  State: draft
  Created: 2020-11-11
</pre>

[TOC]


## Global Architecture

We segregated three main software components: Client, Daemon, and Syncer. 
- The Client drives the swap,
- the Daemon orchestrates the swap protocol execution and
- the Syncer maintains the Protocol state and the Blockchain state in sync  

The figure below represents the general architecture.

![](https://github.com/farcaster-project/RFCs/raw/master/images/arch.png)


The following table summarizes different aspects of each component.

|                 | `swap-client`                    | `swap-daemon`                                        | `chain-syncer`     |
|-----------------|----------------------------------|------------------------------------------------------|--------------------|
| definition   | a program that controls the daemon and display the current state | a program that executes the core protocol in a state machine | a program that talks with a specific blockchain |
| cryptographic keys & secrets | private & public    | public only                                          | public only        |
| client/user  | end-user                         | `swap-client`, counterparty `swap-daemon`                | `swap-daemon`        |
| availability          | present at the start and to sign | mostly online, channel of communication between parties | always online      |
| communicates with | `swap-daemon`                      | `swap-client`, `chain-syncer`, counterparty `swap-daemon` | `swap-daemon`, blockchain |
| transactions    | signs                            | creates all transactions, verifies signatures        | listens for and publishes transactions  |
| protocol-state  | doesn't understand protocol, but can represent its state              | understands the protocol, but can't sign             | doesn't understand protocol |

### Client/daemon segregation rationale
The rationale behind segregating the client and the daemon currently is not for security reasons -- the client signs the transactions received from the daemon blindly, implying full trust.

The client is the only component that has access to secret keys.

The aim of this segregation is to improve flexibility and extensibility added by making the client peripheral to the swap stack, that is, other clients might be created, such as: 
- clients supporting hardware wallets
- mobile applications (that may run the daemon in background or in a private server), 
- heavy- or light-weight desktop GUIs, 
- scripted/automated backend clients (e.g. run by market maker, OTCs etc) 

### Components interaction

Each swap components is represented as a black box that consumes input messages and produces output messages. Each input and output message has a type, components consume certain types of messages only, e.g. the client doesn't consume messages produced by syncers.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/arch-global.png)

It is worth noting that this diagram only show one syncer, but a syncer per blockchain is required.

Below a string diagram formalization of the information flow.
![](https://i.imgur.com/UdxbWui.png)


## Daemon

The Daemon is the central component responsible for orchestrating the protocol execution. 

Daemon's main function is to manage the safe progression of the execution of the cross-chain-swap in response to the intrinsic and extrincic messages it receives -- respectively coming from itself and from the components it directly interacts with. 

The Daemon MUST be fully aware of the complete State of the cross-chain-swap execution. To achieve that it MUST listen to: 
- blockchain events from both chains via Syncers communication,
- swap-counterparty protocol messages via inter-daemon communication, 
- user's instructions via Client communication, and
- self-produced input messages

The Daemon MUST create a constrained runtime environment for executing the protocol, that only permits valid protocol transitions at all times. To achieve that a petrinet model of the protocol may be used to constraint the runtime environment that executes the user's respective swap role in the protocol, by only authorizing firing valid enabled protocol transitions.

### Inter-daemon communication
The Daemon has a bidirectional communication channel with the swap counterparty's daemon. This inter-daemon communication is used to pass messages in the correct order between the swap counterparties, such as the messages needed for:
- agreeing on the global swap parameters,
- safe initialization,
- and safe protocol execution during swap period itself.

### Client-Daemon communication
1. A valid Client Instruction message sent by the Client to the Daemon instructs the daemon to fire a given enabled protocol transition 
2. Daemon consumes Client Instruction message and
3. Daemon fires transitions that are in one-to-one correspondence with Client instructions
4. As a consequence of firing protocol transitions, Daemon's internal Swap state MAY be modified
5. If the Swap State was modified, Daemon MUST update Client by sending a State Digest message to the Client about the new enabled protocol transitions Client MAY next instruct to fire
6. Client then MAY fire any of the new enabled protocol transition and progress on the protocol execution (back to step 1)

The figure below presents a petrinet summarizing client-daemon communication scheme.

![](https://i.imgur.com/PwdlQTF.png)


### Blockchain communication: Syncer-Daemon communication
A Daemon does not interact with a blockchain fullnode directly. 

A Syncer handles 'tasks' requests from a Daemon and MAY produce a set of 'blockchain events' according to the type of 'task' it receives.

### Loopback: self-generated input messages
The daemon MAY generate self-addressed messages. Those messages MAY be used to trigger transitions only based on daemon's state. Those transitions can e.g. be the absence of counter-party communication during a periode of time.

## Syncer
A syncer is specific to a blockchain and can handle a list of 'tasks' directly related to that blockchain. Those 'tasks' will be completed in different manners depending on the blockchain type and/or the blockchain state. The logic inside syncers allow the daemon to abstract a part of the logic needed to interact with a blockchain with a define interface composed of 'tasks' and 'blockchain events'.

### Tasks
A syncer is responsible to handle 'tasks' messages, its inputs. To achieve that goal a syncer is connected to a blockchain, through a full node or equivalent, and uses e.g. RPC calls and 0MQ notification streams to produce 'blockchain events' messages, its outputs.

Tasks MUST follow those rules:
* The same task MUST be publishable multiple times without causing contradictory side effects
* A task published more than once MUST always produce an equivalent set of events
* Tasks MUST have a defined lifetime

### Blockchain Events
Blockchain Events are produced by syncers in response to certain type of 'tasks'. A task MAY produce multiple 'blockchain events'. When a 'task' produces two different set of 'blockchain events' depending on when the 'task' is handled by the syncer those two sets MUST have an equivalent impact on the state at any point in time.

## Client

The client is the only component aware of the user's private keys and acts as a "swap wallet", signing the blockchain transactions and piloting the swap through 'instructions'.

Instruction messages are asynchronous and may fail, the daemon MUST handle missing instructions from a client and MAY modify the Swap state in response.

Client and Daemon MUST have an initialization protocol allowing one or the other to recover from a past Swap state.

Daemon updates Client with State Digest messages to move forward with the swap protocol execution.

Client pass User's instructions to the Daemon. That is, Client MAY at the User's discretion fire one of the possible protocol transitions at each moment in time. A subset of transitions MAY require signed protocol messages and/or signed blockchain transactions as input. After Client's instruction, Daemon MUST undertake the actions associated with the fired transition, and update Swap state accordingly.

## Swap State
The Swap state encodes the step of the protocol execution the user is currently in, and it is handled by the Daemon. For each given state, zero or more transitions are enabled as a function of a list of valid inputs. Inputs can be: (i) blockchain events, (ii) protocol messages, (iii) client instructions, or (iv) daemon loopback messages. When a swap state receives an input, a transition to a new state MAY produce outputs. Outputs can be: (i) syncer tasks, (ii) protocol messages, (iii) client state digest messages, or (iv) daemon loopback messages.

Valid transitions from one state to other states are described by the protocol. Upon Swap state update, daemon MUST update the Client through State Digest messages.

For each swap state, only a subset of inputs is valid. The daemon MUST contextually filter all of its inputs before applying them. Filtering can happen in parallel for every stream of inputs. All filtered stream are then combined, each element of this final stream are applied one by one on the current state.

The first task of the combined input stream MUST be a save on disk. After each transition the swap state MUST be saved on disk as the last checkpoint. The daemon MUST be able to start with a given checkpoint, a saved stream of inputs and a protocol.

The swap state can be viewed as an ordered set of inputs. On a crash the daemon MUST be able to load the last checkpoint and apply the stream of inputs (saved and incoming inputs) uncontained in the loaded checkpoint.

By ensuring that daemon output messages can be replayed safely, like tasks, the work already performed since the last saved checkpoint can be replayed safely.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/input-streams.png)

### Recovery from saved state between components
#### Inter-daemon
Daemon to Daemon communication MUST have a mechanism to reconnect and safely and gracefully recover from the latest saved state. This is the most critical recovery of the protocol as recovering to the wrong state MAY permit counter-party to steal funds.

#### Client-Daemon
Both Daemon and Client MUST implement mechanisms to restablish their connection and safely recover from a previously saved state.

#### Syncer-Daemon
Daemon MUST implement mechanisms to reconnect to Syncer and safely recover from its previously saved state and from the extra information queried from Syncer about events while Daemon was disconnected.