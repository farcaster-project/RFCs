<pre>
  State: draft
  Created: 2021-02-24
</pre>

# 09. Swap State

## Overview

The state of a swap is an aggregate of events generated or received by each system component involved in the protocol execution. This RFC describes the state management of these components.

## Table of Contents

  * [Daemon state](#daemon-state)
  * [State transition](#state-transition)
  * [Datum messages](#datum-messages)
  * [Transcripts](#transcripts)
  * [Transcript state recovery](#transcript-state-recovery)
  * [Checkpoint recovery](#checkpoint-recovery)
    * [Inter-daemon](#inter-daemon)
    * [Client-Daemon-Syncer (same user)](#client-daemon-syncer-same-user)
    * [Syncer-Daemon](#syncer-daemon)

## Daemon state
The daemon encodes the swap state-space as a Petri net. The marking of the Petri net encodes the swap state. Only swap states resulting from a valid protocol execution are included in the state-space. A valid protocol execution respects the state transition rules.

The daemon handles the swap state and its state transitions on behalf of the client and following the client's instructions. The swap state indicates the protocol execution step the user is currently on. For each given state, zero or more state transitions are enabled as a function of a list of time-ordered daemon transcripts. That is, depending on the protocol execution path taken, these are the next available user actions.

## State transition
Upon swap `state transition`, the daemon must update the client through `datum` messages, which in turn provides the resources for the client to create a new `datum` to further command the daemon forward.

The daemon must contextually filter all of its input messages with respect to the interaction component (client, syncer, or counter-party-daemon) they relate to. It writes to `transcript` the messages in the order it processes them, on a single thread.  All filtered streams may then be combined if needed, and each element of this final stream is applied to the current state.

The first task is to save to disk the input messages, creating the execution transcripts. After each transition is applied to the state, the swap state may be saved on disk (if safety critical) as the last checkpoint. The daemon must be able to recover (1) from a given checkpoint (containing the last recorded state), and (2) may recover from a stream of input messages that followed the checkpoint, and the protocol descriptor. The latter recovery mechanism allows recovery to any swap state, but may not be needed, as in the current protocol there are not many safety critical states (namely 2 for each participant's daemon). All other states must be safely derived from these critical states.

The state may be recovered purely on the basis of the transcripts using a fold operation that returns the state. The state recovery process must not produce new messages (= no side-effects).

The swap state can be derived from an ordered set of incoming messages. On a crash, the daemon may be able to load the last checkpoint and apply the stream of inputs (saved and incoming inputs) uncontained in the loaded checkpoint.

By ensuring that daemon output messages can be replayed safely, like tasks, the work already performed since the last saved checkpoint can be replayed safely.

## Datum messages
Datum messages are generic messages --- bidirectionally exchanged by client and daemon --- which allow the construction of types that are needed for protocol progressions, such as a signature or transaction. They convey the information needed to construct the state on the other side. The instantiation of the type is equivalent to placing a token in a petrinet. The marking of the net encodes the full state, and this one token is thus a partial state. 

## Transcripts 
Transcripts are time-ordered, and composed of 
  - Syncer's blockchain `Event` Accordant
  - Syncer's blockchain `Event` Arbitrating
  - Client data bundles, `Bundles`
  - Counter-party Daemon `Protocol` messages
  - Own Daemon `Protocol` messages 
  - Own Daemon Internal-timers, `Loopback`
  - Published task from Daemon to Accordant Syncer, `Task`
  - Published task from Daemon to Arbitrating Syncer, `Task`

The transcripts include incoming messages to the daemon and outgoing messages emitted by the daemon.

The transcripts shall be written from a single thread and have sequential order for each component.

## Transcript state recovery

`L` is the complete set of valid transcripts produced by valid protocol runs and `Σ` is the set of protocol states, the state-space.

There is a function, `recover_state` or `r` that reads all the `transcripts`,

```
                L → Σ
                r(transcripts) = σ
```

and that outputs a protocol state, `σ`.

The recovery operation would take the following form in pseudocode:

``` rust
let recovered_state: State = transcripts.fold(initial, |state, transcript| state.apply(transcript));
```

Although desirable, transcript state recovery may not be essential, as checkpoint recovery is also possible, and easier to implement.

## Checkpoint recovery

`Checkpoint`s must provide all the data to instantiate the types that underlie the state. They are expensive and shall be used only on critical sections.

### Inter-daemon
Any interaction prior to the coins being locked can safely be ignored. Recovery to a state prior to locking funds is handy but optional. 

Before Bitcoin and Monero are locked, inter-daemon may fail with no further issues, and may not need recovery. 

`checkpoint pre-lock`: Both Alice and Bob, just before locking their coins, shall make a checkpoint. 
`checkpoint post-buyprocsig`: Both Alice and Bob shall amend the `checkpoint pre-lock` by concatenating the `buy_procedure_signature` message after it is sent by Bob's to Alice's daemons. 

### Client-Daemon-Syncer (same user)

Daemon assumes Client and Syncer are stateless.

All the interactions from the `buy_procedure_signature` protocol message until the end of the swap are critical. Nonetheless, since it's within a trusted setup composed of its own Daemon, Client, and Syncer setup, it can be safely replayed.

The Daemon keeps track of its state but assumes Client and Syncer are stateless. 

#### Daemon side

Upon Daemon recovery, Daemon must watch for all transactions it knows about through the syncers, in order to detect if the swap evolved since the last checkpoint, before taking action. Thus Daemon must create tasks for watching for transactions, before publishing them, in order to make sure the recovery process is building upon the original run, for example.

#### Client side

The Client has to backup to disk: its own id, swap parameters, and Daemon id, and must have access to secret keys. From this backup, Daemon's messages sent during the recovery process are used to resume the swap, since Daemon is a trusted component. After a crash, a recovering Client must receive Daemon's datum messages, which are parametric on Daemon's swap state. From such messages, the Client may provide the additional signatures to complete the swap, if needed. As such, the Client is mostly stateless.

The data needed for the Client to recover were already stored on Daemon's checkpoints:
  - After `Signed adaptor buy` and `Signed arbitrating lock transactions`, a `checkpoint` is taken on Bob's Daemon side. Thereafter Bob locks the Bitcoin. Note that this corresponds to Bob's `checkpoint pre-lock`, so no new checkpoint is needed.
  - After `Signed adaptor refund` and `Cosign arbitrating cancel` are received by Daemon, and just before locking the Monero, Alice's Daemon makes a `checkpoint`. Again, this corresponds to Alice's `checkpoint pre-lock`, so no new checkpoint is needed.

Both daemon-client and daemon-syncer pairings must implement mechanisms to re-establish their connection.

### Syncer-Daemon

As tasks can be replayed safely, the daemon and syncer do not need any particular recovery mechanism.

Syncer shall recover to the same set of tasks as before crashing.
