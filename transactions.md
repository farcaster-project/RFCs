[![hackmd-github-sync-badge](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA/badge)](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA)

<pre>
  State: draft
  Created: 2020-12-11
</pre>

[TOC]

# Transactions

Dashed outline transactions are transaction created by external wallets, i.e. not the daemon nor the client.

## Bitcoin

This RFC defines six Bitcoin transactions.

![Bitcoin transaction graph](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/btc-transactions.png)

### Pre-lock

The `pre-lock (a)` transaction is an externally created transaction that serves two purposes:

 1. Force the creation of a SegWit UTXO for the `lock (b)` transaction
 2. Not make any assumption on where the funds come from

The transaction can have `n` inputs and `m` outputs but MUST create 1 and only 1 P2WPKH (SegWit v0) UTXO for the given address during initialization. [we miss a ref. on the graph for this lock condition]

The P2WPKH rationale is to have a better support, this can be moved to a SegWit v1 P2TR later when support is added in wallets.

### Lock

The `lock (b)` transaction consumes the SegWit UTXO from `pre-lock (a)` and create a Taproot UTXO `(i)` with the locking script (SegWit v1) `OP_1 0x20 <Q pubkey>`:

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

`TapLeaf cancel script`, the cancel script, is a 2-of-2 multisig with timelock constraint used by the `cancel (d)` transaction.

> It is worth noting that it might be possible to make a cooperative cancel, allowing to not reveal the `TapLeaf script` on-chain for better privacy and smaller fees.

`TapLeaf cancel script`:

```
<num> [TIMEOUTOP]
EQUALVERIFY DROP
<Alice's Ac PubKey> CHECKSIG <Bob's Bc PubKey> CHECKSIGADD m 
NUMEQUAL
```

`[TIMEOUTOP]` is either `CHECKSEQUENCEVERIFY` or `CHECKLOCKTIMEVERIFY`.

> It is worth noting that `CHECKLOCKTIMEVERIFY` will probably not work if we want to support RBF (Replace By Fee)

### Buy

The `buy` transaction is available as soon as the local confirmation security threshold is reached by the buyer.

It consumes the `lock`'s Taproot output `(i)` with just a signature (MuSig2+adaptor), revealing the secret spend key to the counterparty.

### Cancel

The script-path witness that consumes `lock`'s P2TR UTXO:

    <nitems> <len> <input> <len> <script> <len> <c>

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block.

`<input>`:

```
<Bob's Bc signature> <Alice's Ac signature>
```

The `cancel (d)` transaction creates a Taproot UTXO `(ii)` with the locking script (SegWit v1) `OP_1 0x20 <Q' pubkey>`:

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

`TapLeaf punish script`, the punish script, is a single signature (Alice's) with timelock constraint used by the `punish` (f) transaction.

`TapLeaf punish script`:

```
<num> [TIMEOUTOP]
EQUALVERIFY DROP
<Alice's Ap PubKey> CHECKSIGVERIFY
```

### Refund

The `refund` transaction is available as soon as the local confirmation security threshold is reached by the seller [use better definition here than seller].

It consumes the `cancel`'s Taproot output `(ii)` with just a signature (MuSig2+adaptor), revealing the second secret spend key to the counterparty, effectively doing the refund.

### Punish

The script-path witness has the same structure as the `buy` transaction with `<input>`:

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

## Notes on Privacy

Bitcoin transaction must be designed in a way where if Taproot is used with a single signature to spend an output, it MUST not be possible to differentiate between `c` and `d`, and between `e` and `f`.

But in most cases `c` and `d` will be different because `d` will be used with the TapLeaf script.

Also, when a script-path spend is chosen, the script and the control block should look indistinguishable from a common protocol such as Lightning Network to increase the anonymity pool.

## Transaction fee

On the arbitrating blockchain a chain of three transactions is created, the two last transactions must be are created in advance and their fees must be determined at creation.

We describe in this chapter some heuristic to set the fees.

Let's define $t(c, t)$, a function of time returning the fee for a transaction $t$ based on a coeficient $c$.

If $c(\texttt{buy}) = 1$, then $c(\texttt{cancel}) > 1$, such that at time $t$ when `buy` and `cancel` are both available `cancel` should have a better probability to be mined.

It is worth noting that we cannot control the coeficient $c$ between `refund` and `punish` as `punish` is control unilaterraly by Alice.

## RBF (Replace By Fee)

RBF (Replace By Fee) should be integrated into the protocol such that multiple version of some transaction can exist.