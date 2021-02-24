# Swap State

## Daemon state
The daemon encodes the swap state-space as a petrinet. The marking of the petrinet encodes the swap state. Only swap states resulting from a valid protocol execution are included in the state-space.

The daemon handles the swap state and its state transitions on behalf of the client, and following its instructions. The swap state indicates the step on the protocol execution the user is currently in.  For each given state, zero or more state transitions are enabled as a function of a list of time-ordered daemon execution logs. That is, depending on the protocol execution path taken, these are the next available user actions.

## Execution logs
The execution logs includes incoming messages to the daemon and outgoing messages emited by the daemon. 

Incoming: (i) blockchain events, (ii) protocol messages, (iii) client instructions, or (iv) daemon loopback messages. 
Outgoing: (i) syncer tasks, (ii) protocol messages, (iii) client state digest messages, or (iv) daemon loopback messages.

## State transition
Upon swap state transition, daemon must update the client through state digest messages, including proposals for the next client instruction.

For each swap state, only a subset of inputs is valid. The daemon must contextually filter all of its inputs before applying them. Filtering can happen in parallel for every stream of inputs. All filtered streams are then combined, each element of this final stream are applied to the current state.

The first task of the combined input stream must be a save on disk. After each transition the swap state must be saved on disk as the last checkpoint. The daemon must be able to start with a given checkpoint, a saved stream of inputs and a protocol.

The swap state can be viewed as an ordered set of inputs. On a crash the daemon must be able to load the last checkpoint and apply the stream of inputs (saved and incoming inputs) uncontained in the loaded checkpoint.

By ensuring that daemon output messages can be replayed safely, like tasks, the work already performed since the last saved checkpoint can be replayed safely.

## State digests
State digests are messages sent by the daemon to the client. They relay the information needed to construct the client state. The client state determins what is presented to user interface, by exposing the currently valid user instructions. These messages both allow the client to display the correct actions a user can perform and create the necessary instructions consumed daemon to continue the swap protocol.

**Format**:

- Current marking of petri net representation of swap
- Data required for firing a given transition

## The `state_digest` Message
Provides the client with the current swap state digest. By applying the fired transitions to the current petri net of the client, the client can infer which transitions are available to fire.

 1. type: 33790 (`state_digest`)
 2. data:
    - [`u16`: `fired_transitions_len`]
    - [`fired_transitions_len * type`: `fired_transitions`] Vector of # of firings per transition since last state digest
    - [`sha256`: `marking_hash`] Hash of the current marking so that client can verify it applied the transitions correctly
    - [`u16`: `fireable_transition_data_len`]
    - [`fireable_transition_data_len * type`: `fireable_transition_data`] Map of fireable transitions to the data that the client requires to produce the authentication the daemon requires to fire it, such as transaction signatures


## Recover state
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

## Update state
There is a function, `update_state` or `u`,

```
                Σ × L → Σ
                u(σt−1, Δlogs) = σt
```

where `σt−1` is the previous state and `σt` is the final state, and `Δlogs` are the logs between `t-1` and `t`.

The protocol state may be encoded as markings of a petrinet.

In practice the code may be implemented as a fold operation over a stream of events taking a petrinet marking as initial state that internally outputs the new marking (which encodes the new protocol state). Then tap the wire after the fold operation to monitor the state.

## Recovery process between components
The recovery process is different depending the components. Client and daemon are trusted, the prerequisits are not the same as e.g. inter-daemon.

### Inter-daemon
Daemon to daemon communication must have a mechanism to reconnect and gracefully recover from the latest saved state. This is the most critical recovery of the protocol as recovering to the wrong state may permit counter-party to steal funds.

TODO(h4sh3d): elaborate a be more precise

### Client-Daemon
Both daemon and client must implement mechanisms to restablish their connection and safely recover from a previously saved state.

TODO(h4sh3d): elaborate a be more precise

### Syncer-Daemon
As tasks can be replayed safely the daemon and syncer does not need any particular recovery mechanism.
