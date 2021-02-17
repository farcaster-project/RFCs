
<pre>
  State: draft
  Created: 2021-2-17
</pre>

# Introduction

## Overview

Farcaster is software stack implementing an atomic swap protocol designed in such a way that blockchains can be connected to securely exchange assets. The RFCs are as blockchain agnostic as possible and when Bitcoin or Monero blockchains are mentionned, they can be replaced by other blockchains fulfilling the same prerequisites.

We created these RFCs to structure our codebase an maximize potential inter-operability with other project or platform of exchange.

## Table of Contents

[TOC]

## Glossary and Terminology Guide

* #### *Instruction*:
   * A message sent by a *client* intended to instruct the *daemon* on what to do next.

* #### *Daemon*:
   * The piece of software running the swaps protocol.

* #### *Swap*:
   * Swap or *atomic swap*; The execution of a cryptographic protocol between two or more participants to allow an exchange of blockchain assets without involving a third party nor trust in participant.

* #### *State Digest*:
   * A message sent by the *daemon* intended to instruct a *client* on what is the current state of a *swap*.

* #### *Client*:
   * A piece of software used by the user containing the cryptographic keys needed to transfert the assets.

* #### *User*:
   * The end user of the swap software stack.

* #### *Syncer*:
   * Syncer or Chain Syncer; A piece of software bridging a *daemon* and a blockchain. Produces *blockchain events* and listen for *tasks*.

* #### *Blockchain Event*:
   * A message sent by a *syncer* intended to instruct the *daemon* on a subscribed blockchain state change.

* #### *Task*:
   * A message sent by the *daemon* intended to instruct a *syncer* what action to perform or on what blockchain state change to listen.

* #### *Protocol message*:
   * A message sent by the *daemon* intended to the *counter-party daemon*.

* #### *Counter-party daemon*:
   * The 'connected to' daemon fulfilling the complementary *swap role* in a *swap*.

* #### *Swap role*:
   * The role a deamon fulfills during the swap phase. Two roles are available: Alice and Bob.

* #### *Phase*:
   * The phase a deamon plays during a swap. Two ordered phases are available: negociation and swap.

* #### *Swap phase*:
   * The phase a deamon plays after the negociation phase to perform the swap. During this phase the daemon is connected and interact with *clients*, *syncers*, and a *counter-party daemon*.

* #### *Negociation phase*:
   * The phase a deamon plays at start. During this phase, depending on its *Negocation role*, the daemon will start listening for a connection or connects to a counter-party daemon.

* #### *Negociation role*:
   * The role a deamon fulfills during the negociation phase. Two roles are available: Maker and Taker.

* #### *Maker*:
   * A negociation role; The role a deamon can fulfills during the negociation phase. As a maker the daemon will create an offer and wait for an incoming connection.

* #### *Taker*:
   * A negociation role; The role a deamon can fulfills during the negociation phase. As a taker the daemon will display an offer and connects to the associated counter-party daemon.

* #### *Blockchain role*:
   * The protocol describe two blockchain roles: arbitrating and accordant. A blockchain may be compatible with both role, e.g. Bitcoin, some only with the accordant role, e.g. Monero, depending on their on-chain capabilities. A *swap* is always performed between on arbitrating blockchain instance and one accordant blockchain instance.

* #### *Arbitrating blockchain*:
   * A blockchain compatible with the arbitrating blockchain role, e.g. Bitcoin.

* #### *Accordant blockchain*:
   * A blockchain compatible with the accordant blockchain role, e.g. Monero.


