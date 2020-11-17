# Farcaster Architecture
###### tags: `farcaster` `RFC` `architecture`

[TOC]

<pre>
  State: draft
  Created: 2020-11-11
</pre>

## Global Architecture

The figure represents the general architecture.
![](https://github.com/farcaster-project/RFCs/raw/master/images/arch.png)

### Client/daemon segregation rationale

The rationale behind segregating the client and the daemon is not for security reasons at the moment (the client signs the transactions received from the daemon blindly, implying full trust), but for the flexibility and extensibility added.

Other clients can be created: mobile applications (that also run the daemon in background), heavy or light desktop GUIs, or even scripted/automated backends (e.g. in a business environment).

Each pieces can be represented as black boxes receiving inputs and producing outputs.

![](https://github.com/farcaster-project/RFCs/raw/master/images/global-arch.jpg)

''It is worth noting that this diagram only show one syncer, but a syncer per blockchain is required.''

## Daemon

The daemon is the central component responsible for moving the state of the swap accordingly to the received outputs. It has a bidirectional communication channel with the other daemon to coordinate on the global swap parameters and inter-daemon communication during the swap period.

Daemon reacts on client 'instruction' inputs and produces 'instruction availability' listen by the client. A daemon and a client MUST implement a mechanism to reconnect given a previously saved state.

A daemon doesn't interact with a blockchain fullnode directly. A syncer does handle 'job' requests from a daemon and MAY produce a set of 'events' according to the type of 'job'.

Daemon to daemon communication MUST have a mechanism to reconnect given a previously saved state.

## Syncer

A syncer is specific to a blockchain and can handle a list of 'jobs' directly related to that blockchain. Those 'jobs' will be completed in different manners depending on the blockchain type.

### Jobs

A syncer is responsible to handle 'jobs'. To achieve that goal a syncer is connected to a blockchain, through a full node or equivalent, and uses e.g. RPC calls and 0MQ notification streams to produce 'events'.

Jobs MUST follow those rules:
* The same job MUST be publishable multiple times without causing problematic side effects
* A job published more than once MUST always produce the equivalent set of events

### Events

Events are produced by syncers in response to certain type of 'jobs'. A job MAY produce multiple 'events'. When a 'job' produces two different set of 'events' depending on when the 'job' is handled by the syncer those two set MUST be
equivalent.

## Client

The client is the only component aware of the user's private keys and acts as a "swap wallet", signing the blockchain transactions and piloting the swap through 'instructions'.

Instruction availibility messages and instructions are asynchronous and may fail, the daemon MUST handle missing instructions from a client and MAY modify the Swap state in reaction.

Client and Daemon MUST have an initialization protocol allowing one or the other to recover from a past Swap state.

Daemon serves Client the list of available 'instructions' to move forward with the swap protocol execution through 'instruction availibility' messages.

Client pass User's instructions to the Daemon. That is, Client MAY at the User's discretion fire one of the possible protocol transitions at each moment in time. A subset of transitions MAY require signed protocol messages and/or signed blockchain transactions as input. After Client's instruction, Daemon MUST undertake the actions associated with the fired transition, and update Swap state accordingly.

## Swap State

The Swap state encodes the step of the protocol execution the user is currently in, and it is handled by the Daemon. For each given State zero or more transitions are enabled as a function of a list of valid inputs. Inputs can be: (i) events, (ii) inter-daemon messages, or (iii) instructions. When a swap state receives an input, a transition to a new state MAY produce outputs. Outputs can be: (i) jobs, (ii) inter-daemon messages, or (iii) instruction availibity.

Valid transitions from one state to other states are described through a recipe. Upon Swap state update, daemon pushes new subset of available instructions to Client.

After each transition the swap state SHOULD be saved on disk. The daemon SHOULD be able to start with a given state and a given recipe. By insuring that daemon outputs can be replayed safely, like jobs, the work already performed since the last saved state can be replayed safely.

