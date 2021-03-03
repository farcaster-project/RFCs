<pre>
  State: draft
  Created: 2020-11-17
</pre>

# 05. Tasks & Blockchain Events

## Overview
The `syncer` main function is to maintain the protocol state and the blockchain state in sync.

This RFC describe the syncer interface: `tasks` and `blockchain events`. Through this interface a daemon bi-directionally interacts with a blockchain.

## Table of Contents

  * [Tasks](#tasks)
    * [The `abort` Task](#the-abort-task)
    * [The `watch_height` Task](#the-watch_height-task)
    * [The `watch_address` Task](#the-watch_address-task)
    * [The `watch_transaction` Task](#the-watch_transaction-task)
    * [The `broadcast_transaction` Task](#the-broadcast_transaction-task)
  * [Blockchain Events](#blockchain-events)
    * [The `task_aborted` Event](#the-task_aborted-event)
    * [The `height_changed` Event](#the-height_changed-event)
    * [The `address_transaction` Event](#the-address_transaction-event)
    * [The `transaction_confirmations` Event](#the-transaction_confirmations-event)
    * [The `transaction_broadcasted` Event](#the-transaction_broadcasted-event)
    * [Equivalence of Event Sets](#equivalence-of-event-sets)
      * [`block_height` event](#block_height-event)
      * [`transaction_broadcasted` event](#transaction_broadcasted-event)


## Tasks

A syncer is responsible for carrying out assigned `tasks` received through incoming `tasks` messages, its input messages. To achieve that goal a syncer connects to a blockchain, through a full node or equivalent, and uses e.g. RPC calls and 0MQ notification streams to compute and emit `blockchain events` messages, its output messages.

Tasks MUST follow those rules:

* The same task MUST be publishable multiple times without causing contradictory side effects
* A task published more than once MUST always produce an equivalent set of events
* Tasks MUST have a defined lifetime

Tasks are available from any syncer, yet their parameters vary depending on the network. Parameters are provided for Bitcoin and Monero to demonstrate how the protocol would be implemented. Any codebase working with Bitcoin or Monero SHOULD use the following definitions.

This RFC lists available tasks that MUST be supported by syncers, unless noted otherwise, and their respective output messages: `blockchain events`. A task produces zero or more `events` during its lifetime.

Every task is accompanied with an `id`, a positive integer which fits into the 32-bit signed integer space. This integer has no rules on its generation or any pattern required by its usage. Every emitted event has its own `id` field corresponding to the task which caused it.

The `error` event is valid for every single task, and it takes the task's ID and provides an integer `code`, as well as optionally a string `message`.

This document frequently references epochs, whose definition is dependent on the protocol in question. For Bitcoin, Monero, and other blockchain-based systems, the epoch is the block height. For systems without the context of a block height, Unix timestamps SHOULD be used.

### The `abort` Task
Task consists in aborting the task with id `id`.

Data: 
- `id`: the task `id` that shall be aborted

To update a task with id `id`, Syncer must receive an `abort` task aborting with `id` parameter, and subsequently submit a new instance of this task which shall have a new `id`. 

### The `watch_height` Task

`watch_height` asks the syncer for notifications about updates to the blockchain's `height` and associated `block` id. This task MUST be implemented for any coin with a blockchain. This task MAY be implemented for any coin without a blockchain. If it is not implemented, an error event must be sent in response to any attempt to start this task.

Required parameters are:
* `lifetime`: Epoch at which the syncer SHOULD drop this task. Until then, this task MUST be maintained, barring the case where an `abort` task aborts it. 

Parameters may be added to specify which blockchain, in order to support any network utilizing multiple.

### The `watch_address` Task

`watch_address` asks the syncer for notifications about when a specified address is involved in a transaction.

Required parameters are:
* `lifetime`: Epoch at which the syncer SHOULD drop this task. Until then, this task MUST be maintained, barring the case where an `abort` task aborts it.

For Bitcoin, the following additional parameters are defined:
* `address`: The address to watch.
* `from_height`: Previous blocks to scan for transactions sent to the specified address.
Bitcoin also has this task return historical transactions, with a possible rate limit being implementation-dependent.

For Monero, the following parameters:
* `public_spend`: The public spend key of the address.
* `private_view`: The private view key of the address.
* `from_height`: Previous blocks to scan for transactions sent to the specified address.

`address_transaction` events are emitted as response to `watch_address` task.

### The `watch_transaction` Task

`watch_transaction` asks a syncer for updates on the status of a transaction.

Required parameters are:
* `hash`: Transaction hash.
* `confirmation_bound`: Upper bound on the confirmation count until which the syncer should report updates on the block depth of the transaction. This task MUST be maintained until this threshold is reached or until `lifetime` has passed.
* `lifetime`: Epoch at which the syncer SHOULD drop this task. This task MUST be maintained until this threshold is reached or until `confirmation_bound` has been reached. 

Once a transaction is seen by the syncer, and passed the confirmation threshold, a `transaction_confirmations` event is emitted. 

### The `broadcast_transaction` Task

`broadcast_transaction` tells a syncer to broadcast a transaction. The syncer MUST broadcast the transaction, unless the transaction already in the blockchain or in the mempool.

The only parameter is:
* `tx`: The raw transaction in its serialized format.

The blockchain event in response is `transaction_broadcasted` event.

## Blockchain Events
The function of the Syncer is to emit events to accomplish its assigned tasks. Syncer's must emit events targeting the daemon (at a later stage of the project, potentially the client as well). Blockchain Events are produced by syncers in response to certain type of `tasks`. A task MAY produce multiple `blockchain event` messages. The `blockchain event` messages as defined below. 

### The `task_aborted` Event
Event in response to `abort_task` Task, emitted only once.
Data:
* `id`: integer
* `success_abort`: bool

### The `height_changed` Event
`watch_height` task was previously defined. That task emits `height changed` blockchain events. Upon new block arrival and reorgs, a `height_changed` event is emitted. It contains:
* `block`: current block_hash 
* `height`: current blockchain height.

Upon task reception, syncers MUST send a `height_changed` event immediately to signify the current height.

### The `address_transaction` Event
Once a `watch_address` address is involved in a transaction, `address_transaction`  emitted. It contains:
* `hash`: Transaction ID.
* `amount`: Value of the amount sent to the specified address.

Further fields may be defined depending on the asset. Any asset based on a blockchain MUST also have:
* `block`: hash of the block mining the transaction

### The `transaction_confirmations` Event
In response to the `watch_transaction` task.

- Data: 
  * `block`: hash of the block mining the transaction.
  * `confirmations`: Number of blocks after `block`

- Semantics:
    - For transaction not seen on mempool, emit `(0x0, None)`
    - For transaction seen on mempool but not mined, emit `(0x0, Some(0))`
    - For transaction mined, emit `(tx_block, Some(confirmations))` where `tx_block` is the block that the transaction got mined, and `confirmations` is the number of blocks extending `tx_block`
    - When `confirmations` >= `confirmation_bound`, emit `(tx_block, Some(confirmation_bound))`. Here `Task` is considered successfully accomplished and terminates. At that time, transaction is considered final (=irreversible). 

Thus:
```
for each transaction being watched by Syncer:
  for every new block arrival extending the chain that the transaction was mined: 
    if confirmations <= confirmation_bound:
      Syncer emits: `(tx_block, Some(confirmations))`
```

Upon block reorg reverting transaction out of blockchain, emit `(0x0, Some(0))` if transaction shows back in mempool or `(0x0, None)` if it's not in the mempool.

### The `transaction_broadcasted` Event
The `broadcast_transaction` task defined above produces upon successful transaction broadcast a success output message: `transaction_broadcasted` blockchain event.

This message contains:
* `tx`: The raw transaction in its serialized format.
* `success_broadcast`: bool


FIXME: move to state RFC
### Equivalence of Event Sets 

When a task produces two different sets of blockchain events depending on when the task is handled by the syncer those two sets MUST have an equivalent impact on the state of the daemon at any point in time.

Examples follows.

#### `block_height` event
Let's define `X` as the current block height. The task is sent to the syncer, initial plus two events are received, for `X`, `X+1`, and `X+2` new heights. At time `t2` the latest state is for `X+2`. If the daemon crashes at `t1` with height `X+1`, and later restarts, at time `t2`, it MUST had previously received `X+1` as initial event and then `X+2`. And finally if the daemon crashes at time `t2` and restarts, the initial blockchain event MUST contains `X+2`. Sets of events are different but equivalent for the daemon state.

#### `transaction_broadcasted` event
The daemon sends the task at time `t` and receives the successful `transaction broadcasted` event at time `t'`, if the daemon crashes between `t` and `t'`, rebroadcasting the task MUST result in the same successful event, despite the fact that the syncer MAY not broadcast the transaction to the full-node a second time.
