[![hackmd-github-sync-badge](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ/badge)](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ)

<pre>
  State: draft
  Created: 2020-11-17
</pre>

[TOC]

# Jobs

Available jobs are specific to syncers and their parameters can vary depending on the related blockchain.

This RFC lists available jobs that MUST be supported by a Bitcoin syncer and a Monero syncer and their respective outputs: events.

A job produces zero or more events during its lifetime.

A job MUST be reproducible without contradictory side effets.

## Bitcoin Jobs

### Watch Transaction
Bitcoin job `watch transaction` allows a daemon to ask a syncer for getting noticed with a `transaction seen` event.

Upon job reception a syncer MUST check if the job is already completed and return, if not the syncer MUST start a background task until the job is completed.

The job is completed after producing the `transaction seen` event or when reaching `maxlife`  epoch or block height.

#### Parameters

* `txid`: the transaction txid to watch
* `confirmations`: number of block needed before producing the event, if 0 it produces the output when the transaction appears in the full-node mempool
* `maxlife`: an epoch or block height after what the job can be discarded if not already completed, same semantic as Bitcoin CLTV

#### Event

`transaction seen`:

* `txid`: the transaction txid
* `block`: the block hash where the transaction is mined, zero value hash if in mempool


## Monero Jobs

### Watch Wallet
Monero job `watch wallet` allows a daemon to ask a syncer for getting noticed when funds arrived in a specific view key pair.

Upon job reception a syncer MUST scan the last `lastblocks` blocks and return corresponding `funds arrived` events and MUST start a background task until the job is completed.

The job is completed after reaching the `maxlife` epoch or block height.

#### Parameters

* `privateview`: the private view key
* `publicspend`: the public spend key
* `confirmations`: number of block needed before producing the `funds arrived` event, if 0 it produces the event when the transaction appears in the full-node mempool
* `lastblocks`: the number of last blocks to scan for transactions
* `maxlife`: an epoch after what the job can be discarded or a block height, same semantic as Bitcoin CLTV

#### Event

`funds arrived`:

* `txid`: the transaction id
* `amount`: the amount received

## Common

### New height
Job `new height` allows a daemon to ask a syncer for getting noticed on each height change until some block height or some epoch.

Upon job reception a syncer MUST directly send an `height changed` event and MUST start a background task until the job is completed.

The job is completed after reaching the block height or the epoch specified in the `maxlife` parameters.

#### Parameters

* `maxlife`: an epoch after what the job can be discarded or a block height, same semantic as Bitcoin CLTV

#### Event

`height changed`:

* `height`: the new height number
* `block`: the block hash

### Broadcast Transaction
Job `broadcast transaction` allows a daemon to ask a syncer for broadcasting a transaction.

Upon job reception a syncer MUST check if the job is already completed and return, if not the syncer MUST handle the job and return.

The job is completed after producing a `transaction broadcasted` event or a `broadcast failed` event.

#### Parameters

* `rawtx`: the raw transaction to broadcast

#### Event

`transaction broadcasted`:

* `txid`: the transaction txid

`broadcast failed`:

* `error`: an error description