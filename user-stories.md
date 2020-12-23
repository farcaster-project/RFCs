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

### Blockchain: Arbitrating & Accordant

We describe two blockchain roles: (1) Arbitrating and (2) Accordant. The former correspond to the chain where the constrain system lives, e.g. the Bitcoin blockchain, the latter is the accordant chain which has no extra on-chain capabilities requirement.

Those roles are distributed by the outgoing blockchains' capabilities involved in the swap pair, e.g Monero cannot be the arbitrating chain.

### Swap: Alice & Bob

Because of the protocol asymetry property, we describe two swap roles: (1) Alice and (2) Bob.

We define Alice's role as: Alice always move coins from the accordant chain to the arbitrating chain.

We define Bob's role as: Bob always move coins from the arbitrating chain to the accordant chain.

## Phases

We distinguish between two phases: (1) negociation and (2) swap phases. The negociation implies Taker and Maker roles, the swap phase implies Alice and Bob roles.

### Negociation phase

The negociation phase can be done on a forum, with an OTC, within an DEX, etc. This RFC only defines the interface between the negociation phase and the swap phase.

We describe the responsability of the two negociation roles.

#### Maker

The maker start his node in maker mode and register all the parameters an offer requires. The daemon setup an onion service and waits for an incoming connection. When ready, the daemon prints the 'public offer', the public offer contains all the parameters a taker needs to connect. The maker can then distribute the 'public offer' over his prefered channels.

##### Maker offer

A maker offer is composed of:

 * The Arbitrating-Accordant blockchain identifier, e.g. BTC-XMR
 * The arbitrating blockchain asset amount
 * The accordant blockchain asset amount
 * The timelocks duration used during the swap
 * The future maker swap role (Alice or Bob)

##### Maker public offer

A maker public offer is a maker offer plus daemon parameters such as the onion service or other options how to connects to the daemon.

The maker public offer must be easily serializable and in a friendly sharing format.

#### Taker

A taker receives a public offer, visualize it and might accept it. If the taker wants to take the public offer he can try to connects to the maker and start the swap.

#### Results of negociation phase

The negociation is out of scope for this RFC but MUST result in:

 * Daemons connected to eachother
 * Agreement on the following set of parameters
     * Blockchains used as an Arbitrating-Accordant asset pair, e.g. BTC-XMR
     * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
     * Transition from Maker-Taker roles into Alice-Bob roles
     * Time parameters t1 and t2, i.e. protocol timelock parameters

### Swap phase

The swap phase starts for each roles with a set of parameters:

 * Blockchains used as an Arbitrating-Accordant asset pair, e.g. BTC-XMR
 * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
 * Time parameters t1 and t2, i.e. protocol timelock parameters
 * The swap role played by the daemon

plus for Alice's role:

 * The destination Bitcoin address

and for Bob's role:

 * The refund Bitcoin address


#### Steps

We can describe the high level view of the swap phase with four steps:

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