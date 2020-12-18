# User Stories

[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)


<pre>
  State: draft
  Created: 2020-11-11
</pre>

[TOC]


## Roles

### Negociation: Taker & Maker

Taker and maker roles are dissociated from swap roles. They are used in the negociation phase. A Taker can latter be transformed into an Alice or Bob role when moving from the negociation phase into the swap phase, and vice versa.

The Maker role offers a potential trade, the proposal can be a buy or sell for one asset pair, with fixed amounts or ranges, etc. There is no special limitation on what a proposal can be. A Taker can respond on a proposal.

### Swap: Alice & Bob

Because of the protocol asymetry property, we describe two swap roles: (1) Alice and (2) Bob.

### Blockchain: Arbitrating & Accordant

We describe two blockchain roles: (1) Arbitrating and (2) Accordant. The former correspond to the chain where the constrain system lives, e.g. the Bitcoin blockchain, the latter is the accordant chain which has no extra on-chain capabilities requirement.

Those roles are distributed by the outgoing blockchains' capabilities involved in the swap pair, e.g Monero cannot be the arbitrating chain.

Alice MUST always move coins from the accordant chain to the arbitrating chain and Bob MUST always move coins from the arbitrating chain to the accordant chain.

## Phases

We distinguish between two phases: (1) negociation and (2) swap phases. The negociation implies Taker and Maker roles, the swap phase implies Alice and Bob roles.

### Negociation phase

The negociation phase can be done on a forum, with an OTC, within an DEX, etc. This RFC only defines the interface between the negociation phase and the swap phase.

The negociation is out of scope for this RFC but MUST result in agreement on the following set of parameters:

 * Blockchains used as an Unscripted-Scripted asset pair, e.g. XMR-BTC
 * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
 * Time parameters t1 and t2, i.e. protocol timelock parameters
 * (Do we agree also on a set of cryptographic parameter such as secondary base point for zkp, etc. ?)

[Maybe we miss other stuff too]
bitcoin destination address? is it a parameter, not IMO, it's a local parameter for the Unscripted asset.

[Some of the stuff below should be part the of the initialization step in the swap phase.]

### Swap phase

[here we kind of assume that they are already connected in some way to eachother, maybe we should be more specific. also depends on the comm. channel we use between daemons!]

We can discribe the high level view of the swap phase with four steps:

 1. Initialization step
 2. Bitcoin locking step
 3. Monero locking step
 4. Swap step (or exchange step?)

We describe a basic user experience with a atomic swap GUI client for Alice and Bob.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/gui-mocks.png)

#### Alice's role

Alice, who owns accordant blockchain's coins, e.g. Monero, is TODO

##### Reputation problem

Because of the protocol asymetry, Alice always locks her coins later in the swap process, that implies she gets an option to buy without costs. One way to resolve this issue is to introduce a reputation system between participants, but this is hard in a decentralized setup.

The reputation problem is not linked to the negociation role took by Alice's role, if she's a Taker she can cancel for free on any prices and if she's a Maker she can propose any prices and cancel for free if someone tries to take it.

#### Bob's role

Bob, who owns arbitrating blockchain's coins, e.g. Bitcoin, is TODO