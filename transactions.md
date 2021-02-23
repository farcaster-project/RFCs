<pre>
  State: draft
  Created: 2020-12-11
</pre>

# Transactions

## Overview

The protocol implemented in Farcaster is blockchain agnostic, thus a strict list of features is required for the arbitrating blockchain involved in the swap (see [00. Introduction](00-introduction.md) and [01. High Level Overview](01-high-level-overview.md)). This RFC describes a concrete implementation of the protocol with Bitcoin as the arbitrating blockchain and Monero as the accordant blockchain.

We distinguish transactions created and controlled by the protocol itself and external transactions. Dashed outline transactions are transaction created by external wallets, i.e. not the daemon nor the client.

## Table of Contents

  * [Bitcoin](#bitcoin)
    * [Pre-lock](#pre-lock)
    * [Lock](#lock)
    * [Buy](#buy)
    * [Cancel](#cancel)
    * [Refund](#refund)
    * [Punish](#punish)
  * [Monero](#monero)
    * [Lock](#lock)
    * [Spend](#spend)
  * [Miscellaneous](#miscellaneous)
    * [Notes on Privacy](#notes-on-privacy)
    * [Transaction fee](#transaction-fee)
    * [Bitcoin transactions temporal safety](#bitcoin-transactions-temporal-safety)

## Bitcoin

This RFC defines the Bitcoin transactions. These transactions can be constructed with three different approaches:

 * **ECDSA Scripts**, with SegWit v0 outputs and ECDSA signatures
 * **Taproot Schnorr Scripts**, with SegWit v1 outputs and Schnorr signatures and on-chain multi-signature using TapLeaf scripts
 * **Taproot Schnorr MuSig2**, with SegWit v1 outputs and Schnorr signatures and MuSig2 off-chain multi-signature protocol

The latter is the prefered option for privacy but depends on features activation on the Bitcoin chain and MuSig2 protocol. This RFC describe for each transaction the three approaches.

#### Timelocks

`[TIMEOUTOP]` is either `CHECKSEQUENCEVERIFY` or `CHECKLOCKTIMEVERIFY`.

> It is worth noting that `CHECKLOCKTIMEVERIFY` will probably not work if we want to support RBF (Replace By Fee)

#### Bitcoin transaction graph

![Bitcoin transaction graph](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/btc-transactions.png)

### Pre-lock

The `pre-lock (a)` transaction is an externally created transaction that serves two purposes:

 1. Force the creation of a SegWit UTXO for the `lock (b)` transaction
 2. Not make any assumption on where the funds come from

The transaction can have `n` inputs and `m` outputs but MUST create 1 and only 1 P2WPKH (SegWit v0) UTXO `(i)` for the given address during initialization.

The P2WPKH rationale is to have a better support, this can be moved to a (SegWit v1) P2TR later when support is added in wallets.

### Lock

The `lock (b)` transaction consumes the SegWit UTXO `(i)` from `pre-lock (a)` and creates the UTXO `(ii)`.

#### ECDSA Scripts

The `lock (b)` creates a (SegWit v0) UTXO `(ii)` with the locking script:

```
IF
    2 <Alice's Ab PubKey> <Bob's Bb(Ta) PubKey> 2 CHECKMULTISIG
ELSE
    <num> [TIMEOUTOP] DROP
    2 <Alice's Ac PubKey> <Bob's Bc PubKey> 2 CHECKMULTISIG
ENDIF

where
    Ab: Alice's buy key;
    Bb: Bob's buy key;
    Ac: Alice's cancel key;
    Bc: Bob's cancel key; and
    Ta: Alice's adaptor key
```

#### Taproot Schnorr Scripts

The `lock (b)` creates a (SegWit v1) Taproot UTXO `(ii)` with the locking script `OP_1 0x20 <Q pubkey>`:

```
              Q                       | the Taproot tweaked key
              |
    ----------------------
    |                    |
    P           Script Merkle root    | P: an internal key where DLog is unknown
                         |
            /---------------------------\
    TapLeaf buy script        TapLeaf cancel script    | SegWit v1 scripts
```

with `TapLeaf buy script`:

```
<Alice's Ab PubKey> CHECKSIG <Bob's Bb(Ta) PubKey> CHECKSIGADD m 
NUMEQUAL

where
    Ab: Alice's buy key;
    Bb: Bob's buy key; and
    Ta: Alice's adaptor key
```

and `TapLeaf cancel script`, the cancel script, a 2-of-2 multisig with timelock constraint used by the `cancel (d)` transaction.

`TapLeaf cancel script`:

```
<num> [TIMEOUTOP]
EQUALVERIFY DROP
<Alice's Ac PubKey> CHECKSIG <Bob's Bc PubKey> CHECKSIGADD m 
NUMEQUAL

where
    Ac: Alice's cancel key; and
    Bc: Bob's cancel key;
```

#### Taproot MuSig2

The `lock (b)` creates a (SegWit v1) Taproot UTXO `(ii)` with the locking script `OP_1 0x20 <Q pubkey>`:

```
              Q                        | the Taproot tweaked key
              |
    ----------------------
    |                    |
    P           Script Merkle root     | P: the internal key, a MuSig2 setup
                         |
              TapLeaf cancel script    | SegWit v1 script
```

`P`, the internal key, is a MuSig2 setup used by the `buy (c)` transaction for transmitting the adaptor signature:

```
P: Ab + Bb + Ta

where
    Ab: Alice's buy key;
    Bb: Bob's buy key; and
    Ta: Alice's adaptor key
```

`TapLeaf cancel script`, the cancel script, as described in *Taproot Schnorr Scripts*.

### Buy

The `buy` transaction is available as soon as the local confirmation security threshold is reached by Alice.

#### ECDSA Scripts

Consumes the `lock`'s output `(ii)` with:

```
0 <Bob's Bb(Ta) signature> <Alice's Ab signature> TRUE <script>
```

and leaks the adaptor `Ta` on `<Bob's Bb(Ta) signature>` to Bob.

#### Taproot Schnorr Scripts

The script-path witness that consumes `lock`'s P2TR UTXO:

```
<nitems> <len> <input> <len> <script> <len> <c>
```

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block;

where `<script>` is the `TapLeaf buy script`;

and `<input>`:

```
<Bob's Bb(Ta) signature> <Alice's Ab signature>
```

and leaks the adaptor `Ta` on `<Bob's Bb(Ta) signature>` to Bob.

#### Taproot MuSig2

Consumes the `lock`'s Taproot output `(ii)` with one valid signature for `<Q>` the Taproot tweaked key, generated with `<P>` (MuSig2+adaptor), revealing the second secret spend key to the counterparty (Bob), effectively doing the swap.

### Cancel

The `cancel (d)` transaction consumes the `lock`'s output `(ii)` and creates the output `(iii)` used to cancel the swap by one of the participant.

#### ECDSA Scripts

Consumes the `lock`'s output `(ii)` with:

```
0 <Bob's Bc signature> <Alice's Ac signature> FALSE <script>
```

and creates a (SegWit v0) UTXO `(iii)` with the locking script:

```
IF
    2 <Alice's Ar(Tb) PubKey> <Bob's Br PubKey> 2 CHECKMULTISIG
ELSE
    <num> [TIMEOUTOP] DROP
    <Alice's Ap PubKey> CHECKSIG
ENDIF

where
    Ar: Alice's refund key;
    Br: Bob's refund key;
    Ap: Alice's punish key; and
    Tb: Bob's adaptor key
```

#### Taproot Schnorr Scripts

The script-path witness that consumes `lock`'s P2TR UTXO:

```
<nitems> <len> <input> <len> <script> <len> <c>
```

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block.

and `<input>`:

```
<Bob's Bc signature> <Alice's Ac signature>
```

Creates a Taproot UTXO `(iii)` with the locking script (SegWit v1) `OP_1 0x20 <Q' pubkey>`:


```
              Q'                      | the Taproot tweaked key
              |
    ----------------------
    |                    |
    P'          Script Merkle root    | P': an internal key where DLog is unknown
                         |
            /---------------------------\
  TapLeaf refund script        TapLeaf punish script    | SegWit v1 scripts
```

with `TapLeaf refund script`:

```
<Alice's Ar(Tb) PubKey> CHECKSIG <Bob's Br PubKey> CHECKSIGADD m
NUMEQUAL

where
    Ar: Alice's refund key;
    Br: Bob's refund key; and
    Tb: Bob's adaptor key
```

and `TapLeaf punish script`, the punish script, a single signature (Alice's) with timelock constraint used by the `punish (f)` transaction.

`TapLeaf punish script`:

```
<num> [TIMEOUTOP]
EQUALVERIFY DROP
<Alice's Ap PubKey> CHECKSIGVERIFY

where
    Ap: Alice's punish key;
```

#### Taproot MuSig2

The script-path witness that consumes `lock`'s P2TR UTXO is the same as described in *Taproot Schnorr Scripts*.

Creates a Taproot UTXO `(iii)` with the locking script (SegWit v1) `OP_1 0x20 <Q' pubkey>`:

```
              Q'                       | the Taproot tweaked key
              |
    ----------------------
    |                    |
    P'          Script Merkle root     | P': the internal key, a MuSig2 setup
                         |
              TapLeaf punish script    | SegWit v1 script
```

`P'`, the internal key, is a MuSig2 setup used by the `refund (e)` transaction for transmitting the second adaptor signature:

```
P': Ar + Br + Tb

where
    Ar: Alice's refund key;
    Br: Bob's refund key; and
    Tb: Bob's adaptor key
```

`TapLeaf punish script`, the punish script, as described in *Taproot Schnorr Scripts*.

### Refund

The `refund (e)` transaction is available as soon as the local confirmation security threshold is reached by Bob.

#### ECDSA Scripts

Consumes the `cancel (d)`'s output `(iii)` with:

```
0 <Bob's Br signature> <Alice's Ar(Tb) signature> FALSE <script>
```

and leaks the adaptor `Tb` on `<Alice's Ar(Tb) signature>` to Alice.

#### Taproot Schnorr Scripts

The script-path witness that consumes `cancel`'s P2TR UTXO:

```
<nitems> <len> <input> <len> <script> <len> <c>
```

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block;

where `<script>` is the `TapLeaf refund script`;

and `<input>`:

```
<Bob's Br signature> <Alice's Ar(Tb) signature>
```

and leaks the adaptor `Tb` on `<Alice's Ar(Tb) signature>` to Alice.

#### Taproot MuSig2

Consumes the `cancel`'s Taproot output `(iii)` with one valid signature for `<Q'>` the Taproot tweaked key, generated with `<P'>` (MuSig2+adaptor), revealing the second secret spend key to the counterparty (Alice), effectively doing the refund.

### Punish

The `punish (f)` transaction consumes `cancel (d)`'s output `(iii)` and transfer the funds to Alice and is available as soon as the timelock is passed.

#### ECDSA Scripts

Consumes the `cancel`'s output `(iii)` with:

```
<Alice's Ap signature> FALSE <script>
```

#### Taproot Schnorr Scripts & Taproot MuSig2

The script-path witness that consumes `cancel`'s P2TR UTXO:

```
<nitems> <len> <input> <len> <script> <len> <c>
```

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block;

where `<script>` is the `TapLeaf punish script`;

and `<input>`:

```
<Alice's Ap signature>
```

## Monero

Two external Monero transactions are defined: (a) the `lock` transaction and (b) the `spend` transaction. The spend transaction consumes the protocol created output from the `lock` transaction by complying with the condition `(i)`.

![Monero transaction graph](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/xmr-transactions.png)

> It is worth noting that all Monero transactions can be handled by external wallet. More precisely the latter by importing the private keys into an external software.

### Lock

The Monero lock transaction (a) is performed by an external wallet and MUST send the full negotiated amount in the address attached to (i). The transaction MUST NOT be broadcasted too early in the protocol, otherwise funds will be lost.

The condition (i) is defined during the protocol initialization phase as a shared secret between the two participants as:

    k_s = k_s^a + k_s^b (mod l)    | the private spend key of the address
    k_v = k_v^a + k_v^b (mod l)    | the private view key of the address

`k_v` is shared and known by both participants.

### Spend

The Monero spend transaction (b) allows the final owner of the funds to move them into an address without revealing the private view key to anyone else for better anonymity. This step is not necessary from a security point of view but required for good privacy.

The spend transaction has to follow the minimum Monero output age policy (10 blocks).

## Miscellaneous

### Notes on Privacy

Bitcoin transaction must be designed in a way where if Taproot and MuSig2 are used with a single signature to spend an output, it MUST not be possible to differentiate between `c` and `d`, and between `e` and `f`.

But in most cases `c` and `d` will be different because `d` will be used with the TapLeaf script.

Also, when a script-path spend is chosen, the script and the control block should look indistinguishable from a common protocol such as Lightning Network to increase the anonymity pool.

### Transaction fee

On the arbitrating blockchain a chain of three transactions is created, the two last transactions must be are created in advance and their fees must be determined at creation.

We describe in this chapter some heuristic to set the fees.

Let's define `f(tx, c)`, a function returning the fee over time for a transaction `tx` based on a coeficient `c`.

If `c(buy) = 1`, then `c(cancel) > 1`, such that at time `t` when `buy` and `cancel` are both available `cancel` has a higher probability to be mined.

It is worth noting that we cannot control the coeficients `c` between `refund` and `punish` as `punish` is control unilaterraly by Alice.

#### Replace By Fee (RBF)

Replace By Fee (RBF) should be integrated into the protocol such that multiple version of some transaction can exist and transactions can be cooperatively bumped to get into the blockchain within the temporal safety window.

### Bitcoin transactions temporal safety

In the swap protocol publishing transactions to the bitcoin mempool (=transaction pool) reveals secret keys that are required to sweep the monero wallet. Therefore it is paramount to carefully evaluate the temporal safety bounds of publishing transactions, as they may:

 - be raced by valid and pre-signed protocol's transactions that are de facto double-spends or 
 - reverted by blockchain reorgs

#### Protocol

Here we offer a simple protocol that may be used for publishing each transaction to the Bitcoin blockchain without putting funds at risk.

The swap participant has a temporal safety parameter, `delta_irreversible`, in blocks. If a transaction is mined at block `t`, at `t + delta_irreversible`, the participant assumes that her transaction is irreversible.

##### Funding

After the atomic swap protocol initialization successfully completes, Bob can publish the funding transaction. Bob publishes the funding transaction, that is later mined at block height `t(funding)`. 

At `t(funding) + delta_irreversible` Alice assumes Bob's transaction is irreversible, and may move on with the protocol execution. That is, Alice can publish her buy transaction. 

##### Buy

However Alice should not wait too long for publishing the buy transaction as the cancellation path may become available, and then a race condition (double spend) between the swap (buy transaction) and the cancellation (cancel transaction) paths becomes possible. 

The safety parameter to prevent race conditions (=double spends) is `delta_race`, in blocks.

The cancellation path becomes valid at time `t(funding) + delta(cancel)$`. Thus Alice has to publish her buy transaction the latest at 

    t(funding) + delta(cancel) - delta_race


In sum, Alice's safety window to publish the buy transaction is from 

    t(funding) + delta_irreversible

to

    t(funding) + delta(cancel) - delta_race

where

    delta_irreversible < delta_race < delta(cancel)

A reasonable guesstimate:

    delta_race ~= 10 x delta_irreversible


Note that `delta_irreversible` and `delta_race` must be proportional to the amount transacted. Additionally `delta_race` is proportional to mempool congestion.

The same logic applies on the cancelation path.

##### Cancel

Cancel transaction should be published as soon as it becomes valid, that is, after 

    t(funding) + delta(cancel)

The block height at which cancel transaction is mined is `t(cancel)`

##### Refund

Refund transaction safety window for publication 

    t(cancel) + delta_irreversible

to

    t(cancel) + delta(punish) - delta_race

It can be argued that the higher bound of the refund transaction window can be extended, not deliberately and safely, but in emergency cases where Bob could not be online. The refund transaction must be published before the punish transaction gets mined, as they consume the same output from the cancel transaction.

The counter argument is that if Bob publishes the refund transaction then Alice can retrieve the secret key and will get the monero, and might as well get the bitcoin if she wins the race.

While if she only gets the bitcoin, and the monero stays locked, Bob could setup a new swap to unlock the already locked monero (no cost for Alice) and propose a nice reward to Alice for revealing her private key. That is a fairly graceful failure.

##### Punish

Punish transaction should be published as soon as it becomes valid, that is, after `t(cancel) + delta(punish)`