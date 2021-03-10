# Swap State

## Daemon state
The daemon encodes the swap state-space as a Petri net. The marking of the Petri net encodes the swap state. Only swap states resulting from a valid protocol execution are included in the state-space. A valid protocol execution respects the state transition rules.

The daemon handles the swap state and its state transitions on behalf of the client and following its instructions. The swap state indicates the step on the protocol execution the user is currently in.  For each given state, zero or more state transitions are enabled as a function of a list of time-ordered daemon transcripts. That is, depending on the protocol execution path taken, these are the next available user actions.

## State transition
Upon swap `state transition`, the daemon must update the client through `datum` messages, which in turn provides the resources for client to create new `datum` to further command the daemon forward.

The daemon must contextually filter all of its input messages, such as, this message relates to syncer, that to client etc. It writes to `transcript` the messages in the order it processes them, on a single thread.  All filtered streams may then combined if needed, and each element of this final stream are applied to the current state.

The first task is to save to disk the input messages, creating the execution transcripts. After each transition is applied to state, the swap state must be saved on disk as the last checkpoint. The daemon must be able to recover (1) from a given checkpoint (containing the last recorded state), and (2) a stream of input messages that followed the checkpoint, and the protocol descriptor. 

The state may be recovered purely on the basis of the transcripts using a fold operation that returns the last markings. The state recovery process must not produce new messages.

The swap state can be derived from an ordered set of incoming messages. On a crash, the daemon must be able to load the last checkpoint and apply the stream of inputs (saved and incoming inputs) uncontained in the loaded checkpoint.

By ensuring that daemon output messages can be replayed safely, like tasks, the work already performed since the last saved checkpoint can be replayed safely.

## Datum messages
Datum messages are generic messages bidirectionally exchanged by client and daemon, that allow the construction of types that are needed for protocol progression, such as a signature or transaction. They convey the information needed to construct the state on the other side. The instantiation of the type is equivalent to placing a token in a petrinet. The marking of the net encodes the full state, and this one token is thus a partial state. 

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

### Checkpoint recovery

`Checkpoint` must provide all the data to instantiate the types that underlie the state. They are expensive and shall be used only on critical sections.

`Checkpoint id` includes the `transcript index`, from when the checkpoint was set if transcripts are on.

Recovering from a checkpoint means `initial` from the code above shall be replaced by `checkpoint`, and `transcripts` should be read from `transcript index` on.

## Recovery process between components

The above mechanism with transcripts or checkpoints shall be used for recovery on different parts of the components that require proper state recovery for safety reasons. 

A less rigid approach might suffice for less critical parts.

Any interaction prior to the coins being locked can be safely ignored. Recovery prior to locking funds is handy but optional. 

### Inter-daemon
Before Bitcoin and Monero are locked, inter-daemon may fail with no further issues, and may not need recovery. 

`checkpoint pre-lock`: Both Alice and Bob, just before locking their coins, shall make a checkpoint. 
`checkpoint post-buyprocsig`: Both Alice and Bob shall amend the `checkpoint pre-lock` by concatenating the `buy_procedure_signature` message, after its sent by Bob's to Alice's daemons. 

After that message the critical daemon-to-daemon communication is over. 

Daemon to daemon communication shall have a mechanism to reconnect and gracefully recover from the checkpoints recommended above. 


### Client-Daemon

#### Daemon side

After `Signed adaptor buy` and `Signed arbitrating lock transactions`, a `checkpoint` is taken on Bob's Daemon side. Thereafter Bob locks the Bitcoin. Note that this corresponds to Bob's `checkpoint pre-lock`, so no new checkpoint needed.

After `Signed adaptor refund` and `Cosign arbitrating cancel` are received by Daemon, and just before locking the Monero, Alice's Daemon makes a `checkpoint`. Again, this corresponds to Alice's `checkpoint pre-lock`, so no new checkpoint needed.

#### Client side
The client has to checkpoint its own id, swap parameters and daemon id, and must have access to secret keys. From that it must be able to use the daemon to recover its own state, since daemon is a trusted component.

Thereafter no more checkpoints for the client.

All the interaction from `buy_procedure_signature` until the end of swap are critical, but must be safely re-playable within a trusted daemon, client and syncer setup.

Both daemon and client must implement mechanisms to re-establish their connection and Daemon helps Client to safely recover from a previous state.

### Syncer-Daemon

As tasks can be replayed safely, the daemon and syncer do not need any particular recovery mechanism. 

Syncer shall recover to the same set of tasks as before crashing.
