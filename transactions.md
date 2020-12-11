[![hackmd-github-sync-badge](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA/badge)](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA)

<pre>
  State: draft
  Created: 2020-12-11
</pre>

[TOC]

# Transactions

Dashed outline transactions are transaction created outside by external wallets.

## Bitcoin

This RFC defines six Bitcoin transactions.

![Bitcoin transaction graph](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/btc-transactions.png)

### Pre-lock

The `pre-lock` transactoin (a) is an externally created transaction that serves two purposes:

 1. Force the creation of a SegWit UTXO for the `lock` transaction
 2. Not make any assumption on where the funds come from

The transaction can have `n` inputs and `m` outputs but MUST create 1 and only 1 P2WPKH (SegWit v0) UTXO for the given address during initialization. [we miss a ref. on the graph for this lock condition]

The P2WPKH rational is too have a better support, this can be moved to a SegWit v2 P2TR latter when support is added in wallets.

### Lock

The `lock` transaction (b) consumes the SegWit UTXO from `pre-lock` and create a Taproot UTXO (i) with the locking script (SegWit v1) `OP_1 0x20 <Q pubkey>`:

```
              Q                        | the Taproot tweaked key
              |
    ----------------------
    |                    |
    P           Script Merkle root     | P: the internal key, MuSig2 setup
                         |
                   TapLeaf script      | script: the time-locked refund script
```

`P`, the internal key, is a MuSig2 setup used by the `buy` (c) transaction for transmitting the adaptor signature.

`TapLeaf script`, the refund script, is a 2-of-2 multisig with timelock contrain used by the `cancel` (d) transaction.

> It is worth noting that it might be possible to make a cooperative refund, allowing to not reveal the `TapLeaf script` on-chain for better privacy and smaller fees.

`TapLeaf script`:

```
<num> [TIMEOUTOP]
EQUALVERIFY DROP
<Alice's PubKey> CHECKSIG <Bob's PubKey> CHECKSIGADD m 
NUMEQUAL
```

`[TIMEOUTOP]` is either `CHECKSEQUENCEVERIFY` or `CHECKLOCKTIMEVERIFY`.

### Buy

The `buy` transaction is available as soon as the local confirmation security threshold is reached by the buyer.

It consumes the `lock`'s Taproot output with just a signature (MuSig2+adaptor), revealing the secret spend key to the counterparty.

### Cancel

The script-path witness that consume `lock`'s P2TR UTXO:

    <nitems> <len> <input> <len> <script> <len> <c>

with `<input>`: an input that fulfills the spending conditions set by `<script>`, and `<c>` the control block.

`<input>`:

```
<Bob's signature> <Alice's signature>
```

### Refund

TODO

### Punish

TODO

### Privacy concerns

Bitcoin transaction must be designed in a way where if Taproot is used with a single signature to spend an output, it MUST not be possible to differanciate between `c` and `d`, and between `e` and `f`.

But in most cases `c` and `d` will be different because `d` will be used with the TapLeaf script.

## Monero

Two external Monero transactions are defined: (a) the `lock` transaction and (b) the `spend` transaction. The spend transaction consumes the protocol created output from the `lock` transaction by complying with the condition (i).

![Monero transaction graph](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/xmr-transactions.png)

### Lock

The Monero lock transaction (a) is performed by an external wallet and MUST send the full negociated amount in the address attached to (i). The transaction MUST NOT be broadcasted too early in the protocol, otherwise funds will be lost.

The codition (i) is defined during the protocol initialization phase as a shared secret between the two participants as:

    k_s = k_s^a + k_s^b (mod l)    | the private spend key of the address
    k_v = k_v^a + k_v^b (mod l)    | the private view key of the address

`k_v` is shared and known by both participants.

### Spend

The Monero spend transaction (b) allow the final owner of the funds to move them into an address with nobody else knowledge of the private view key for better anonymity. This step is not necessary from a security point of view but required for good privacy.