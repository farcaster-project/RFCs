<pre>
  State: draft
  Created: 2021-02-19
</pre>

# 01. High Level Overview

## Introduction

This RFC describes the high level concepts associated with the protocol such as roles and phases of a swap. Roles apply to blockchains and participants during the phases, phases split the protocol into two parts: negotiation and swap.

## Table of Contents

  * [Phases](#phases)
    * [Negotiation phase](#negotiation-phase)
    * [Swap phase](#swap-phase)
  * [Roles](#roles)
    * [Blockchain: Arbitrating & Accordant](#blockchain-arbitrating--accordant)
    * [Negotiation: Taker & Maker](#negotiation-taker--maker)
    * [Swap: Alice & Bob](#swap-alice--bob)
  * [Reputation asymmetry](#reputation-asymmetry)

## Phases

This RFC describes two phases: a negotiation and a swap phase. The swap phase always follows the negotiation phase. The negotiation phase is used to bootstrap the swap parameters common to each participant and connect the swap daemons, this via the concept of a *public offer*. A public offer is a list of parameters describing what the swap will look like if executed. It contains the assets exchanged, how to connect the daemons, and the protocol specific parameters, see [10. Public Offer](./10-public-offer.md).

### Negotiation phase

The negotiation phase can be done on a forum, with an OTC, within a DEX, etc. This RFC and [02. User Stories](./02-user-stories.md) define the interface between the two phases: negotiation and swap, and proposes a minimalistic negotiation protocol.

It is worth mentioning that the negotiation phase proposed in [02. User Stories](./02-user-stories.md) does not contain any negotiation mechanism, e.g. price, amounts, etc. It would be more accurately described as a *discover, connect, and accept* mechanism. Thus this mechanism can be extended and external matching engines could implement a negotiation protocol. This negotiation protocol could then end up with the described *connect and accept* protocol.

During the negotiation phase the *discovery* between participants is done through *public offers*. A public offer is created by one participant and shared, others can then parse and, if interested, accept the public offer. The acceptance is expressed by connecting to the node specified in the offer.

This design is created such that services can be built on top of it, in a purely peer-to-peer network where offers are distributed among "swappers", or oriented as a publicly available service à la OTC where the offer always comes from the same entity, or even inside more elaborated network within DEXs. Everything is compatible with the proposed design since the only constraint is the interface between the negotiation and the swap phases.

### Swap phase

The swap phase always follows the negotiation, daemons are connected to each other and ready to start the swap protocol. Below, this RFC describes the roles participants can have during every phases, a role transition is operated by each participant between the phase transition, this role transition is an important parameter that compose the public offer and must be agreed on.

## Roles

During the phases, different roles are distributed to participants and blockchains. The protocol itself treats blockchains differently based on their on-chain features. Each blockchain instance receives a role during the entire protocol execution.

### Blockchain: Arbitrating & Accordant

We describe two blockchain roles: (1) Arbitrating and (2) Accordant. The former corresponds to the chain where the constraint system lives, e.g. the Bitcoin blockchain in a BTC-XMR setup, the latter is the accordant chain which has no extra on-chain capabilities requirement, e.g. the Monero blockchain in a BTC-XMR setup.

Those roles are determined by the outgoing blockchains' capabilities involved in the swap pair, e.g Monero cannot be the arbitrating chain. Most of the time the choice is dictated by necessity, e.g. BTC-XMR, but with e.g. BTC-ETH there's flexibility in the assignment of their roles, since both are capable of either. However, it is worth noting that there may exist better protocols for exchanging assets in such chain pairings.

### Negotiation: Taker & Maker

To allow the interconnection among participants and set the parameters of a trade, we describe two negotiation roles: (1) Taker and (2) Maker. The former will browse the public offers and the latter will produce them.

Taker and maker roles are dissociated from swap roles. They are used in the negotiation phase. A taker can later be transformed into an Alice or a Bob role when moving from the negotiation phase into the swap phase, and vice versa.

The maker role offers a trade, via the public offer. Its proposal sets the amounts, the asset pair, and what role each participant takes in the swap. There is no special limitation on what a proposal can be in theory. The maker then sends the offer to potential takers.

A taker inspects an offer and decides whether or not to engage in the offered trade.

### Swap: Alice & Bob

Emergent from the protocol's asymmetry, we identify two swap roles: (1) Alice and (2) Bob. Each plays a different and thus complementary game, together composing the entire atomic swap protocol.

Alice always moves coins from the accordant chain to the arbitrating chain. In other words, Alice sells accordant assets in return for arbitrating assets.

Bob always moves coins from the arbitrating chain to the accordant chain. In other words, Bob sells arbitrating assets in return for accordant assets.

## Reputation asymmetry

Due to the protocol's asymmetry, Alice always locks her coins later in the swap process, implying that she gets an option to buy without cost. One way to resolve this issue is to introduce a reputation system between participants, but this is hard in a decentralized setup.

The reputation asymmetry is not linked to the negotiation role assumed by Alice's daemon: If she's a Taker she can cancel for free on any prices and if she's a Maker she can propose any prices and cancel for free if someone tries to take it.

More broadly, the "one has to lock funds first" problem is not due to this protocol nor its asymmetry, but concerns all layer-1 protocols based on multilateral lock/refund primitives.
