<pre>
  State: draft
  Created: 2020-11-17
</pre>

# 05. Tasks & Blockchain Events

## Overview
The `syncer`'s main function is to maintain the protocol state and the blockchain state in sync.

This RFC describes the syncer interface: `tasks` and `blockchain events`. Through this interface, a daemon bi-directionally interacts with a blockchain.

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

A syncer is responsible for carrying out assigned `tasks` received through incoming `tasks` messages, which are its sole input messages from the `daemon`. To achieve that goal a syncer connects to a blockchain, through a full node or equivalent, and uses e.g. RPC calls and 0MQ notification streams to compute and emit `blockchain events` messages, its output messages.

Tasks MUST follow those rules:

 * The same task MUST be publishable multiple times without causing contradictory side effects
 * A task published more than once MUST always produce an equivalent set of events
 * Tasks MUST have a defined lifetime

Tasks are available from any syncer, yet their parameters vary depending on the network. Parameters are provided for Bitcoin and Monero to demonstrate how the protocol would be implemented. Any codebase working with Bitcoin or Monero SHOULD use the following definitions.

This RFC lists available tasks that MUST be supported by syncers, unless noted otherwise, and their respective output messages: `blockchain events`. A task produces zero or more `events` during its lifetime.

Every task is accompanied by an `id`, a positive integer that fits into the 32-bit signed integer space. This integer has no rules on its generation or any pattern required by its usage. Every emitted event has its own `id` field corresponding to the task which caused it.

The `error` event is valid for every single task, and it takes the task's ID and provides an integer `code`, as well as optionally a string `message`.

This document frequently references epochs. The definition of epochs is dependent on the protocol in question. For Bitcoin, Monero, and other blockchain-based systems, the epoch is the block height. For systems without the context of a block height, Unix timestamps SHOULD be used.

### The `abort` Task

Aborts the task with id `id`.

 1. type: ? (`abort`)
 2. data:
    - [`i32`: `id`]: The task identifier that shall be aborted

To update a task with identifier `id`, Syncer must receive an `abort` task aborting with `id` parameter, and subsequently submit a new instance of this task which shall have a new `id`.

### The `watch_height` Task

`watch_height` asks the syncer for notifications about updates to the blockchain's `height` and associated `block` id. This task MUST be implemented for any coin with a blockchain. This task MAY be implemented for any coin without a blockchain. If it is not implemented, an `error` event must be sent in response to any attempt to start this task.

Required parameters are:

 1. type: ? (`watch_height`)
 2. data:
    - [`i32`: `id`]: The task identifier
    - [`u64`: `lifetime`]: Epoch at which the syncer SHOULD drop this task. Until then, this task MUST be maintained, barring the case where an `abort` task aborts it.

Parameters may be added to specify which blockchain, in order to support any network utilizing multiple.

### The `watch_address` Task

`watch_address` asks the syncer for notifications about when a specified address is involved in a transaction.

Required parameters are:

 1. type: ? (`watch_address`)
 2. data:
    - [`i32`: `id`]: The task identifier
    - [`u64`: `lifetime`]: Epoch at which the syncer SHOULD drop this task. Until then, this task MUST be maintained, barring the case where an `abort` task aborts it.
    - [`bool`: `include_tx`]: Include the full serialized raw transaction if true. Do not include if false.

For Bitcoin, the following additional parameters are defined:

 2. data:
    - [`u16`: `address_len`]
    - [`address_len * byte`: `address`]: The address to watch.
    - [`u64`: `from_height`]: Previous blocks to scan for transactions sent to the specified address.

Bitcoin also has this task return historical transactions, with a possible rate limit being implementation-dependent.

For Monero, the following parameters:

 2. data:
    - [`u16`: `point_len`]
    - [`point_len * byte`: `public_spend`]: The public spend key of the address.
    - [`u16`: `scalar_len`]
    - [`scalar_len * byte`: `private_view`]: The private view key of the address.
    - [`u64`: `from_height`]: Previous blocks to scan for transactions sent to the specified address.

`address_transaction` events are emitted as a response to the `watch_address` task.

### The `watch_transaction` Task

`watch_transaction` asks a syncer for updates on the status of a transaction.

Required parameters are:

 1. type: ? (`watch_transaction`)
 2. data:
    - [`i32`: `id`]: The task identifier
    - [`sha256`: `hash`]: Transaction hash.
    - [`u16`: `confirmation_bound`]: Upper bound on the confirmation count until which the syncer should report updates on the block depth of the transaction. This task MUST be maintained until this threshold is reached or until `lifetime` has passed.
    - [`u64`: `lifetime`]: Epoch at which the syncer SHOULD drop this task. This task MUST be maintained until this threshold is reached or until `confirmation_bound` has been reached.

Once a transaction is seen by the syncer, and passed the confirmation threshold, a `transaction_confirmations` event is emitted.

### The `broadcast_transaction` Task

`broadcast_transaction` tells a syncer to broadcast a transaction. The syncer MUST broadcast the transaction unless the transaction already in the blockchain or the mempool.

 1. type: ? (`broadcast_transaction`)
 2. data:
    - [`i32`: `id`]: The task identifier
    - [`u16`: `tx_len`]
    - [`tx_len * byte`: `tx`]: The raw transaction in its serialized format.

The blockchain event in response is the `transaction_broadcasted` event.

## Blockchain Events

The function of the Syncer is to emit events to accomplish its assigned tasks. Syncer's must emit events targeting the daemon (at a later stage of the project, potentially the client as well). Blockchain Events are produced by syncers in response to certain types of `tasks`. A task MAY produce multiple `blockchain event` messages. The `blockchain event` messages are as defined below.

### The `task_aborted` Event

Event in response to `abort` task, emitted only once.

 1. type: ? (`task_aborted`)
 2. data:
    - [`i32`: `id`]: The task identifier aborted
    - [`i32`: `success_abort`]: The status code, return 0 if task is aborted successfully

### The `height_changed` Event

`watch_height` task was previously defined. That task emits `height_changed` blockchain events. Upon new block arrival and reorgs, a `height_changed` event is emitted. It contains:

 1. type: ? (`height_changed`)
 2. data:
    - [`i32`: `id`]: The task identifier that emits this event
    - [`sha256`: `block`]: Current block hash
    - [`u64`: `height`]: Current blockchain height.

Upon task reception, syncers MUST send a `height_changed` event immediately to signify the current height.

### The `address_transaction` Event

Once a `watch_address` address is involved in a transaction, `address_transaction` is emitted. It contains:

 1. type: ? (`address_transaction`)
 2. data:
    - [`i32`: `id`]: The task identifier that emits this event
    - [`sha256`: `hash`]: Transaction ID.
    - [`u64`: `amount`]: Value of the amount sent to the specified address.
    - [`u16`: `tx_len`]
    - [`tx_len * byte`: `tx`]: Include the raw transaction in its serialized format if the corresponding `watch_address` task's `include_tx` field is set to `true`, use `0x0`, if it is set to false.

Further fields may be defined depending on the asset. Any asset based on a blockchain MUST also have:

 2. data:
    - [`sha256`: `block`]: Hash of the block mining the transaction

### The `transaction_confirmations` Event

In response to the `watch_transaction` task.

 1. type: ? (`transaction_confirmations`)
 2. data:
    - [`i32`: `id`]: The task identifier that emits this event
    - [`sha256`: `block`]: Hash of the block mining the transaction.
    - [`i32`: `confirmations`]: Number of blocks after `block`, return `-1` for `None`

 - Semantics:
    - For transaction not seen on mempool, emit `(0x0, None)`
    - For transaction seen on mempool but not mined, emit `(0x0, Some(0))`
    - For transaction mined, emit `(tx_block, Some(confirmations))` where `tx_block` is the block that the transaction got mined, and `confirmations` is the number of blocks extending `tx_block`
    - When `confirmations` >= `confirmation_bound`, emit `(tx_block, Some(confirmation_bound))`. Hereafter, `Task` is considered successful and terminates. At that time, transaction is considered final (=irreversible).

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

 1. type: ? (`transaction_broadcasted`)
 2. data:
    - [`i32`: `id`]: The task identifier that emits this event
    - [`u16`: `tx_len`]
    - [`tx_len * byte`: `tx`]: The raw transaction in its serialized format.
    - [`i32`: `success_broadcast`]: The status code, return 0 if broadcast is successful

