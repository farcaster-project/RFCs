<pre>
  State: draft
  Created: 2021-2-17
</pre>

# 00. Introduction

## Overview

Farcaster is a software stack implementing an atomic swap protocol designed in such a way that blockchains can be connected to securely exchange assets. The RFCs are as blockchain agnostic as possible and when Bitcoin or Monero blockchains are mentioned, they can be replaced by other blockchains fulfilling the same prerequisites.

We created these RFCs to structure our codebase and maximize potential inter-operability with other projects. If you want to contribute to the code base or the RFCs you may want to study the RFCs closely first to get a solid idea of what Farcaster and the tech behind it are all about.

## Table of Contents

  * [Blockchain Prerequisites](#blockchain-prerequisites)
  * [What is an Atomic Swap](#what-is-an-atomic-swap)
  * [Glossary and Terminology Guide](#glossary-and-terminology-guide)

## Blockchain Prerequisites

The Farcaster protocol can be implemented for many blockchains. We chose Bitcoin and Monero because we see great potential in this particular pairing, but others can be added in parallel or later. The protocol is asymmetric and requires different features for the two exchanged assets' blockchains. The [01. High Level Overview](./01-high-level-overview.md) RFC describe two blockchain roles: *Arbitrating* and *Accordant*.

The *Arbitrating* blockchain requires on-chain features. We describe them as:

* Ability to create a safe money-lock
  * In most UTXO based blockchains this is similar to the ability to chain unbroadcasted transactions and define temporal transaction validity, i.e. the ability to create a refund transaction that becomes valid after some time delay. On blockchains with more expressive  on-chain features, e.g. smart contracts, this prerequisite should always be fulfilled.
* ECDSA or Schnorr based signatures for transactions
  * The secret transmission is based on adaptor signatures and this project only focuses on ECDSA and Schnorr-based adaptor signatures. It might not be impossible to create adaptor signatures for other signature schemes but this is out-of-scope of this project.

On the other hand, the *Accordant* blockchain does not have any on-chain requirements. However, it is worth noting that if the two blockchains use different cryptographic curves a cross-group discrete logarithm zero-knowledge proof is required to secure the protocol. This project provides one between `curve25519` and `secp256k1`.

## What is an Atomic Swap?

An atomic swap is generally a protocol involving two or more blockchain-based assets. The protocol itself ensures that all participants involved in the atomic swap are treated correctly if they behave according to the protocol. If participants follow the protocol, they will at the end either obtain the exchanged assets or get refunded, whereas if they deviations from the protocol, the counterparty participant can punish them.

Such protocols can be designed with cryptography and sometimes game theory. Purely cryptography-based protocols generally impose more constraints on exchanged assets' blockchains but do not rely on game theory to fully secure the trade. Protocols involving game theory, like this one, generally have some punishment mechanism to incentivize the correct behavior described in the protocol.

**Let's explain how the game theory works in Farcaster:**

The main goal of a swap is to have a successful trade. But imagine if one participant disappears or acts maliciously after all have locked their money. In such a case, the swap must be cancelled and participants must be refunded. The refund process in this protocol is heavily linked to one participant called the arbitrating seller. He only has one choice: refund his money. By doing it he also refunds others' money. If the arbitrating seller doesn't react the refund cannot happen for others. Now the arbitrating seller can be honest or malicious: in the case he's honest he will behave according to the protocol and everyone will be refunded, but if he's malicious he can choose to do nothing, letting everybody hanging.

To incentivize the arbitrating seller to do the action of refunding his money, a game takes place: if he doesn't react, after a determined time frame, the other participant can take his money for free, meaning he gets punished.

## Glossary and Terminology Guide

* #### *Accordant blockchain*:
   * A blockchain compatible with the accordant blockchain role, e.g. Monero.

* #### *Arbitrating blockchain*:
   * A blockchain compatible with the arbitrating blockchain role, e.g. Bitcoin.

* #### *Blockchain event*:
   * A message sent by a *syncer* intended to instruct the *daemon* on a subscribed blockchain state-change.

* #### *Blockchain role*:
   * The protocol describes two blockchain roles: arbitrating and accordant. A blockchain may be compatible with both roles, e.g. Bitcoin, and some only with the accordant role, e.g. Monero, depending on their on-chain capabilities. A *swap* is always performed between an arbitrating blockchain instance and one accordant blockchain instance.

* #### *Client*:
   * A piece of software of the *user* containing their cryptographic keys needed to transfer the assets.

* #### *Counter-party daemon*:
   * The 'connected to' *daemon* fulfilling the complementary *swap role* in a *swap*.

* #### *Daemon*:
   * The piece of software running the swaps protocol.

* #### *Instruction*:
   * A message sent by a *client* intended to instruct the *daemon* on what to do next.

* #### *Maker*:
   * A *negotiation role*; The role a *daemon* can fulfill during the negotiation phase. As a maker, the *daemon* will create an offer and wait for an incoming connection.

* #### *Negotiation phase*:
   * The phase a *daemon* plays at the start. During this phase, depending on its *Negotiation role*, the *daemon* will start listening for a connection or connects to a counter-party daemon.

* #### *Negotiation role*:
   * The role a *daemon* fulfills during the negotiation phase. Two roles are available: Maker and Taker.

* #### *Offer*:
   * An offer is created by a *maker* and sets the arbitrating/accordant blockchain pair, the amounts of assets exchanged, the *timelock parameters* used for the swap, and the swap roles for the maker and taker in the following swap phase.

* #### *Phase*:
   * The phase a *daemon* plays during a swap. Two ordered phases are available: negotiation and swap.

* #### *Protocol message*:
   * A message sent by the *daemon* intended to the *counter-party daemon*.

* #### *Public offer*:
   * A public offer is derived from the maker offer and contains *daemon*'s information such as public IP/Tor onion service. A public offer is serializable in a shareable friendly format. It is the maker's responsibility to propagate the public offer on a channel based on his preferences.

* #### *Swap*:
   * Swap or *atomic swap*; The execution of a cryptographic protocol between two or more participants to allow an exchange of blockchain assets without involving a third party nor trust in participants.

* #### *Swap phase*:
   * The phase a *daemon* plays after the negotiation phase to perform the swap. During this phase, the *daemon* is connected and interacts with *clients*, *syncers*, and a *counter-party daemon*.

* #### *Swap role*:
   * The role a *daemon* fulfills during the swap phase. Two roles are available: the accordant seller and the arbitrating seller.

* #### *Syncer*:
   * Syncer or Chain Syncer; A piece of software bridging a *daemon* and a blockchain. Produces *blockchain events* and listens for *tasks*.

* #### *Taker*:
   * A *negotiation role*; The role a *daemon* can fulfill during the negotiation phase. As a taker, the *daemon* will display an offer and connects to the associated counter-party *daemon*.

* #### *Task*:
   * A message sent by the *daemon* intended to instruct a *syncer* what action to perform, or on what blockchain state-change to listen.

* #### *Timelock parameters*:
   * Two arbitrating blockchain timelocks are needed in the protocol. These form the Timelock parameters and are part of the *public offer*.

* #### *User*:
   * The end-user of the swap software stack.

