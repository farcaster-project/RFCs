[![hackmd-github-sync-badge](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ/badge)](https://hackmd.io/0UBnjLo3QzWx_ReejLHgYQ)

<pre>
  State: draft
  Created: 2020-11-17
</pre>

# Jobs

Available jobs are specific to syncers and their parameters can vary depending on the related blockchain.

This RFC lists available jobs that MUST be supported by a Bitcoin syncer and a Monero syncer and their respective outputs: events.

A job can produce zero or more events during its lifetime.

A job MUST be reproducible without undesirable side effets.

## Bitcoin Jobs

### Watch Transaction

Lifetime: finite, the job is completed after producing the first event or when reaching maxlife epoch.

Parameters:

* txid: the transaction txid to watch
* confirmations: number of block needed before producing the output, if 0 it produces the output when the transaction appears in the full node mempool
* maxlife: an epoch after what the job can be discarded if not already completed

Output:

* event: transaction seen

### Broadcast Transaction

Lifetime: finite, the job is completed after producing the first event.

Parameters:

* rawtx: the raw transaction to broadcast

Output:

* event: transaction broadcasted

or

* event: transaction broadcast failed

## Monero Jobs

### Watch Wallet

Lifetime: finite, the job is completed after reaching the maxlife epoch.

Parameters:

* privateview: the private view key
* publicspend: the public spend key
* confirmations: number of block needed before producing the output, if 0 it produces the output when the transaction appears in the full node mempool
* maxlife: an epoch after what the job can be discarded

Output:

* event: funds arrived

### Broadcast Transaction

Lifetime: finite, the job is completed after producing the first event.

Parameters:

* rawtx: the raw transaction to broadcast

Output:

* event: transaction broadcasted

or

* event: transaction broadcast failed
