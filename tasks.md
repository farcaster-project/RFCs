[![hackmd-github-sync-badge](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ/badge)](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ)

<pre>
  State: draft
  Created: 2020-11-17
</pre>

[TOC]

# Tasks

Tasks are available from any syncer, yet their parameters vary depending on the network. Parameters are provided for Bitcoin and Monero to demonstrate how the protocol would be implemented. Any codebase working with Bitcoin or Monero SHOULD use the following definitions.

This RFC lists available tasks that MUST be supported by syncers, unless noted otherwise, and their respective outputs: events. A task produces zero or more events during its lifetime.

Every task is accompanied with an `id`, a positive integer which fits into the 32-bit signed integer space. This integer has no rules on its generation or any pattern required by its usage. Every emitted event has its own `id` field corresponding to the task which caused it.

The `error` event is valid for every single task, and it takes the task's ID and provides an integer `code`, as well as optionally a string `message`.

This document frequently references epochs, whose definition is dependent on the protocol in question. For Bitcoin, Monero, and other blockchain-based systems, the epoch is the block height. For systems without the context of a block height, Unix timestamps SHOULD be used.

## 0: Watch Height

`watch_height` asks the syncer for notifications about updates to the blockchain's height. This task MUST be implemented for any coin with a blockchain. This task MAY be implementeted for any coin without a blockchain. If it is not implemented, an error event must be sent in response to any attempt to start this task.

Required parameters are:
* `lifetime`: Epoch at which the syncer SHOULD drop this task. Until then, barring another instance of this task, this task MUST be maintained. When another instance appears, syncers MAY only keep the most recent one.

Parameters may be added to specify which blockchain, in order to support any network utilizing multiple.

When the height changes, a `height_changed` event is emitted. It contains:
* `height`: Current blockchain height.

Upon task reception, syncers MUST send a `height_changed` event immediately to signify the current height.

## 1: Watch Address

`watch_address` asks the syncer for notifications about when a specified address receives funds.

Required parameters are:
* `confirmations`: Confirmation threshold to wait for before producing an event. If 0, the event is created as soon as a transaction is seen.
* `lifetime`: Epoch at which the syncer SHOULD drop this task. Until then, this task MUST be maintained.

For Bitcoin, the following additional parameters are defined:
* `address`: The address to watch.
* `past_blocks`: Previous blocks to scan for transactions sent to the specified address.
Bitcoin also has this task return historical transactions, with a possible rate limit being implementation-dependent.

For Monero, the following parameters:
* `public_spend`: The public spend key of the address.
* `private_view`: The private view key of the address.
* `past_blocks`: Previous blocks to scan for transactions sent to the specified address.

Once a transaction is seen by the syncer, and passed the confirmation threshold, a `transaction_received` event is emitted. It contains:
* `tx`: Transaction ID.
* `amount`: Value of the amount sent to the specified address.

Further fields may be defined depending on the coin. Any coin based on a blockchain MUST also have:

* `block`: The hash of the block which contains this transaction. If the transaction has yet to be included in a block, a zero value hash is used.

## 2: Watch Transaction

`watch_transaction` asks a syncer for updates on the status of a transaction.

Required parameters are:
* `hash`: Transaction hash.
* `confirmations`: Confirmation threshold to wait for before producing an event. If 0, the event is created as soon as a transaction is seen.
* `confirmation-bound`: Upper bound on the confirmation count until which the syncer should report updates on the block depth of the transaction. This task MUST be maintained until this threshold is reached or until until `lifetime` has passed.
* `lifetime`: Epoch at which the syncer SHOULD drop this task. This task MUST be maintained until this threshold is reached or until until `confirmation-bound` has been reached. 

Once a transaction is seen by the syncer, and passed the confirmation threshold, a `transaction_seen` event is emitted. It contains:
* `confirmations`: Current confirmation threshold.
Further fields may be defined depending on the coin. Any coin based on a blockchain MUST also have:
* `block`: The hash of the block which contains this transaction. If the transaction has yet to be included in a block, a zero value hash is used.

## 3: Broadcast Transaction

`broadcast_transaction` tells a syncer to broadcast a transaction. The syncer MUST broadcast the transaction, even if it was already broadcasted.

The only parameter is:
* `tx`: The raw transaction in its serialized format.

# Blockchain Events

## Equivalent blockchain event sets
Let's define a `new height` task, this task produces `height changed` blockchain events upon new block and reorgs. Let's define $X$ as the current block height. The task is sent to the syncer, initial plus two events are received, for $X$, $X+1$, and $X+2$ new heights. At time $t$ the latest state is for $X+2$. If the daemon crashes at $X+1$ and restarts, at time $t$ it MUST have received $X+1$ as initial event and $X+2$, the latest state is the same. And finally if the daemon crashes at time $t$ and restarts, the initial blockchain event MUST contains $X+2$. Sets of events are different but equivalent for the daemon state.

Let's define a `broadcast transaction` task for tasks that have side effects, this job produces as a success output a `transaction broadcasted` blockchain event. The daemon sends the task at time $t$ and recieves the successful `transaction broadcasted` event at time $t'$, if the daemon crashes between $t$ and $t'$, rebroadcasting the task MUST result in the same successful event, despite the fact that the syncer will not broadcast the transaction to the full-node a second time.