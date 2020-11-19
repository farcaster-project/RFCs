# Farcaster Architecture

[![hackmd-github-sync-badge](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA/badge)](https://hackmd.io/vTCjO2-ySr6SB7ObuJMhnA)

<pre>
  State: draft
  Created: 2020-11-11
</pre>

###### tags: `RFC` `architecture`

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
- scripted/automated backend clients (e.g. in a business environment) 

Each swap components is represented as a black box that consumes input messages and produces output messages.

![](https://github.com/farcaster-project/RFCs/raw/master/images/global-arch.jpg)

It is worth noting that this diagram only show one syncer, but a syncer per blockchain is required.

## Daemon

The Daemon is the central component responsible for orchestrating the protocol execution. 

Daemon's main function is to manage the safe progression of the execution of the cross-chain-swap in response to the extrinsic and intrinsic messages it receives -- respectively coming from itself and from the components it directly interacts with. 

The Daemon MUST be fully aware of the complete State of the cross-chain-swap execution. To achieve that it MUST listen to: 
- on-chain events from both chains via Syncers communication,
- swap-counterparty protocol messages via inter-daemon communication, 
- user's instructions via Client communication

The Daemon MUST create a constrained runtime environment for executing the protocol, that only permits valid protocol transitions at all times. To achieve that a petrinet model of the protocol may be used to create a runtime environment that executes the user's respective role in the protocol, by only authorizing firing valid enabled protocol transitions.

### Inter-daemon communication
The Daemon has a bidirectional communication channel with the swap counterparty's daemon. This inter-daemon communication is used to pass signed messages in the correct order between the swap counterparties, such as the messages needed for:
- agreeing on the global swap parameters, 
- safe initialization, 
- and safe protocol execution during swap period itself.

### Client-Daemon communication
1. Client fires one of the enabled transitions, by sending a valid Client Instruction message to the Daemon 
2. Daemon consumes Client Instruction message
3. Daemon executes Client's instruction
4. Daemon's internal Swap state MAY be modified by instruction execution
5. if the Swap State was modified, Daemon MUST update Client by sending an Enabled Transitions message to the Client about the new enabled protocol transitions Client MAY next fire
6. Client then MAY fire any of the new enabled protocol transition and progress on the protocol execution (back to step 1)

A valid Client Instruction message fires an enabled protocol transition in the Daemon

### Blockchain communication: Syncer-Daemon communication
A Daemon does not interact with a blockchain fullnode directly. A 

A Syncer handles 'job' requests from a Daemon and MAY produce a set of 'events' according to the type of 'job' it receives.

## Recovery from saved state between components
### Inter-daemon
Daemon to Daemon communication MUST have a mechanism to reconnect and safely and gracefully recover from the latest saved state. This is the most critical recovery of the protocol as recovering to the wrong state MAY permit counter-party to steal funds.

### Client-Daemon
Similarly, both Daemon and Client MUST implement mechanisms to restablish their connection and safely recover from a previously saved state.

### Syncer-Daemon
Again, Daemon MUST implement mechanisms to reconnect to Syncer and safely recover from its previously saved state and from the extra information received from Syncer that occured while Daemon was disconnected.

## Syncer
A syncer is specific to a blockchain and can handle a list of 'jobs' directly related to that blockchain. Those 'jobs' will be completed in different manners depending on the blockchain type.

### Jobs

A syncer is responsible to handle 'jobs'. To achieve that goal a syncer is connected to a blockchain, through a full node or equivalent, and uses e.g. RPC calls and 0MQ notification streams to produce 'events'.

Jobs MUST follow those rules:
* The same job MUST be publishable multiple times without causing consequential side effects
* A job published more than once MUST always produce an equivalent set of events

### Events

Events are produced by syncers in response to certain type of 'jobs'. A job MAY produce multiple 'events'. When a 'job' produces two different set of 'events' depending on when the 'job' is handled by the syncer those two set MUST be
equivalent.

## Client

The client is the only component aware of the user's private keys and acts as a "swap wallet", signing the blockchain transactions and piloting the swap through 'instructions'.

Instruction availability messages and instructions are asynchronous and may fail, the daemon MUST handle missing instructions from a client and MAY modify the Swap state in response.

Client and Daemon MUST have an initialization protocol allowing one or the other to recover from a past Swap state.

Daemon serves Client the list of available 'instructions' to move forward with the swap protocol execution through 'instruction availibility' messages.

Client pass User's instructions to the Daemon. That is, Client MAY at the User's discretion fire one of the possible protocol transitions at each moment in time. A subset of transitions MAY require signed protocol messages and/or signed blockchain transactions as input. After Client's instruction, Daemon MUST undertake the actions associated with the fired transition, and update Swap state accordingly.

## Swap State

The Swap state encodes the step of the protocol execution the user is currently in, and it is handled by the Daemon. For each given State zero or more transitions are enabled as a function of a list of valid inputs. Inputs can be: (i) events, (ii) inter-daemon messages, or (iii) instructions. When a swap state receives an input, a transition to a new state MAY produce outputs. Outputs can be: (i) jobs, (ii) inter-daemon messages, or (iii) instruction availibity.

Valid transitions from one state to other states are described through a recipe. Upon Swap state update, daemon pushes new subset of available instructions to Client.

After each transition the swap state SHOULD be saved on disk. The daemon SHOULD be able to start with a given state and a given recipe. By ensuring that daemon outputs can be replayed safely, like jobs, the work already performed since the last saved state can be replayed safely.

