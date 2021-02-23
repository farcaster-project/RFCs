
<pre>
  State: draft
  Created: 2021-2-17
</pre>

# Introduction

## Overview

Farcaster is software stack implementing an atomic swap protocol designed in such a way that blockchains can be connected to securely exchange assets. The RFCs are as blockchain agnostic as possible and when Bitcoin or Monero blockchains are mentionned, they can be replaced by other blockchains fulfilling the same prerequisites.

We created these RFCs to structure our codebase an maximize potential inter-operability with other projects. If you want to contribute to the code base or the RFCs you may want to read all of them first to get a solid idea of what Farcaster is all about and the tech behind it.

## Table of Contents

  * [Blockchain Prerequisites](#blockchain-prerequisites)
  * [What is an Atomic Swap](#what-is-an-atomic-swap)
  * [Glossary and Terminology Guide](#glossary-and-terminology-guide)

## Blockchain Prerequisites

The protocol can be implemented for many blockchains, we chose Bitcoin and Monero because we see great potential, but others can be added in parallel or latter. The protocol is asymetric and requires different features for the two assests' exchanged blockchains. The [01. High Level Overview](./01-high-level-overview.md) RFC describe two blockchain roles: *Arbitrating* and *Accordant*.

The *Arbitrating* blockchain requires on-chain features. We describe them as:

 * Ability to create a safe money-lock
     * In most UTXO based blockchain this is similar to the ability to chain unbroadcasted transaction and define temporal transaction validity, in other words the ability to create a refund transaction that become valid after some time. In more on-chain featured blockchain, e.g. with smart contracts, this prerequisite should always be fulfilled.
 * ECDSA or Schnorr based signatures use in transaction
     * The secret transmission is based on adaptor signatures and this project only focus on ECDSA and Schnorr based adaptor signature. It might not be impossible to create adaptor signature for other signature scheme but this is out-of-scope of this project.

On the other hand the *Accordant* blockchain does not have any on-chain requirement. However, it is worth noting that if the two blockchains use different cryptographic curves a cross-group discrete logarithm zero-knowledge proof is required to secure the protocol. This project provides one between `curve25519` and `secp256k1`.

## What is an Atomic Swap

An atomic swap is generally a protocol involving two or more blockchain based assets. The protocol itself ensure that all participants involved in the atomic swap, if they behave according the to protocol, i.e. if they follow the protocol, will either obtain at the end the exchanged assets or get refunded.

Such protocol can be designed with cryptography and sometime game theory. Pure cryptographicly based protocols generally impose more constraints on assets' exchanged blockchains but do not rely on game theory to fully secure the trade. Protocols involving game theory, as this one, generally have some punishment mechanism to incentivize the correct behavior described in the protocol.

**Let's explain how the game theory works in Farcaster:**

The main goal of a swap is to have a successful trade. But imagine if one participant disapear or acts malicious after all locked their money, the swap must be cancelled and participants must be refunded. The refund process in this protocol is heavily linked to one participant called Bob, he only has one choice: refund his money, by doing it he also refund others' money. If Bob doesn't react the refund cannot happen for others. Now Bob can be honnest or malicious, in the case he's honnest he will behave according to the protocol and everyone will be refunded, but if he's malicious he can choose to do nothing, letting everybody hanging.

To incentivize Bob to do the action of refunding his money a game take place: if he doesn't react, after a determined time frame, the other participant can take his money for free, i.g. he gets punished.

## Glossary and Terminology Guide

* #### *Accordant blockchain*:
   * A blockchain compatible with the accordant blockchain role, e.g. Monero.

* #### *Arbitrating blockchain*:
   * A blockchain compatible with the arbitrating blockchain role, e.g. Bitcoin.

* #### *Blockchain event*:
   * A message sent by a *syncer* intended to instruct the *daemon* on a subscribed blockchain state change.

* #### *Blockchain role*:
   * The protocol describe two blockchain roles: arbitrating and accordant. A blockchain may be compatible with both role, e.g. Bitcoin, some only with the accordant role, e.g. Monero, depending on their on-chain capabilities. A *swap* is always performed between on arbitrating blockchain instance and one accordant blockchain instance.

* #### *Client*:
   * A piece of software used by the user containing the cryptographic keys needed to transfert the assets.

* #### *Counter-party daemon*:
   * The 'connected to' daemon fulfilling the complementary *swap role* in a *swap*.

* #### *Daemon*:
   * The piece of software running the swaps protocol.

* #### *Instruction*:
   * A message sent by a *client* intended to instruct the *daemon* on what to do next.

* #### *Maker*:
   * A negotiation role; The role a deamon can fulfills during the negotiation phase. As a maker the daemon will create an offer and wait for an incoming connection.

* #### *Negotiation phase*:
   * The phase a deamon plays at start. During this phase, depending on its *Negotation role*, the daemon will start listening for a connection or connects to a counter-party daemon.

* #### *Negotiation role*:
   * The role a deamon fulfills during the negotiation phase. Two roles are available: Maker and Taker.

* #### *Offer*:
   * An offer is created by a *maker* and sets the arbitrating/accordant blockchain pair, the amounts of assets exchanged, the *timelock parameters* used for the swap, and the swap roles for the maker and taker in the following swap phase.

* #### *Phase*:
   * The phase a deamon plays during a swap. Two ordered phases are available: negotiation and swap.

* #### *Protocol message*:
   * A message sent by the *daemon* intended to the *counter-party daemon*.

* #### *Public offer*:
   * A public offer is derived from the maker offer and contains *daemon*'s information such as public IP/Tor onion service. A public offer is serializable in a shareable friendly format. It is the maker responsability to propagate the public offer on channel based on his own preferences.

* #### *State Digest*:
   * A message sent by the *daemon* intended to instruct a *client* on what is the current state of a *swap*.

* #### *Swap*:
   * Swap or *atomic swap*; The execution of a cryptographic protocol between two or more participants to allow an exchange of blockchain assets without involving a third party nor trust in participant.

* #### *Swap phase*:
   * The phase a deamon plays after the negotiation phase to perform the swap. During this phase the daemon is connected and interact with *clients*, *syncers*, and a *counter-party daemon*.

* #### *Swap role*:
   * The role a deamon fulfills during the swap phase. Two roles are available: Alice and Bob.

* #### *Syncer*:
   * Syncer or Chain Syncer; A piece of software bridging a *daemon* and a blockchain. Produces *blockchain events* and listen for *tasks*.

* #### *Taker*:
   * A negotiation role; The role a deamon can fulfills during the negotiation phase. As a taker the daemon will display an offer and connects to the associated counter-party daemon.

* #### *Task*:
   * A message sent by the *daemon* intended to instruct a *syncer* what action to perform or on what blockchain state change to listen.

* #### *Timelock parameters*:
   * Two arbitrating blockchain timelocks are needed in the protocol, they are denominated under the name Timelock parameters and are part of the *public offer*.

* #### *User*:
   * The end user of the swap software stack.




