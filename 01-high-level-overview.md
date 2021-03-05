<pre>
  State: draft
  Created: 2021-02-19
</pre>

# 01. High Level Overview

## Introduction

This RFC describes the high level concepts associated with the protocol such as roles and phases of a swap. Roles apply to blockchains and participants during the phases, phases split the protocol into two parts: negotiation and swap.

## Table of Contents

  * [Phases](#phases)
  * [Roles](#roles)
    * [Blockchain: Arbitrating & Accordant](#blockchain-arbitrating--accordant)
    * [Negotiation: Taker & Maker](#negotiation-taker--maker)
    * [Swap: Alice & Bob](#swap-alice--bob)
  * [Reputation asymmetry](#reputation-asymmetry)

## Phases

Farcaster describes two phases: a negotiation and a swap phase. The swap phase always follows the negotiation phase. The negotiation phase is used to bootstrap the swap parameters common to each participant and connect the swap daemons.

## Roles

During the phases, different roles are distributed to participants and blockchains. The protocol itself treats blockchains differently based on their on-chain features. Each blockchain instance receives a role during the entire protocol execution.

### Blockchain: Arbitrating & Accordant

We describe two blockchain roles: (1) Arbitrating and (2) Accordant. The former corresponds to the chain where the constraint system lives, e.g. the Bitcoin blockchain in a BTC-XMR setup, the latter is the accordant chain which has no extra on-chain capabilities requirement, e.g. the Monero blockchain in a BTC-XMR setup.

Those roles are distributed by the outgoing blockchains' capabilities involved in the swap pair, e.g Monero cannot be the arbitrating chain. Most of the time the choice is dictated, e.g. BTC-XMR, but with e.g. BTC-ETH we could argue what role each blockchain should receive, as both can endorse both roles. However, it is worth noting there may exist better protocols to exchange assets in such setups.

### Negotiation: Taker & Maker

Taker and maker roles are dissociated from swap roles. They are used in the negotiation phase. A taker can later be transformed into an Alice or Bob role when moving from the negotiation phase into the swap phase, and vice versa.

The maker role offers a trade. Its proposal sets the amounts, the asset pair, and what role each participant takes in the swap. There is no special limitation on what a proposal can be in theory. The taker then sends the offer to whom might accept it.

A taker visualizes an offer and chooses to try the trade or not.

### Swap: Alice & Bob

Because of the protocol asymmetry property, we describe two swap roles: (1) Alice and (2) Bob. Each will play a different and thus complementary game, together composing the entire atomic swap protocol.

Alice always moves coins from the accordant chain to the arbitrating chain. In other words, Alice sells accordant assets for arbitrating assets.

Bob always moves coins from the arbitrating chain to the accordant chain. In other words, Bob sells arbitrating assets for accordant assets.

## Reputation asymmetry

Because of the protocol asymmetry, Alice always locks her coins later in the swap process, implying that she gets an option to buy without costs. One way to resolve this issue is to introduce a reputation system between participants, but this is hard in a decentralized setup.

The reputation asymmetry is not linked to the negotiation role assumed by Alice's daemon: If she's a Taker she can cancel for free on any prices and if she's a Maker she can propose any prices and cancel for free if someone tries to take it.

