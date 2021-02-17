
[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)

<pre>
  State: draft
  Created: 2020-11-11
</pre>

# User Stories / High Level Protocol

## Overview

This RFC describe the roles and phases of a swap and presents simple user stories, mockups and basic cli example.

## Table of Contents

[TOC]

## Roles

### Negotiation: Taker & Maker

Taker and maker roles are dissociated from swap roles. They are used in the negotiation phase. A taker can later be transformed into an Alice or Bob role when moving from the negotiation phase into the swap phase, and vice versa.

The maker role offers a potential trade, the proposal can be a buy or sell for one asset pair, with fixed amounts or ranges, etc. There is no special limitation on what a proposal can be. A taker can respond on a proposal.

### Blockchain: Arbitrating & Accordant

We describe two blockchain roles: (1) Arbitrating and (2) Accordant. The former corresponds to the chain where the constraint system lives, e.g. the Bitcoin blockchain, the latter is the accordant chain which has no extra on-chain capabilities requirement.

Those roles are distributed by the outgoing blockchains' capabilities involved in the swap pair, e.g Monero cannot be the arbitrating chain.

### Swap: Alice & Bob

Because of the protocol asymmetry property, we describe two swap roles: (1) Alice and (2) Bob.

We define Alice's role as: Alice always move coins from the accordant chain to the arbitrating chain.

We define Bob's role as: Bob always move coins from the arbitrating chain to the accordant chain.

## Phases

We distinguish between two phases: (1) negotiation and (2) swap phases. The negotiation implies taker and maker roles, the swap phase implies Alice and Bob roles.

### Negotiation phase

The negotiation phase can be done on a forum, with an OTC, within an DEX, etc. This RFC only defines the interface between the negotiation phase and the swap phase.

We describe the responsibility of the two negotiation roles.

#### Maker

The maker starts her node in maker mode and registers all the parameters an offer requires. The daemon creates an onion service and waits for an incoming connection. When ready, the daemon prints the 'public offer'. The public offer contains all the parameters a taker needs to connect. The maker can then distribute the 'public offer' over her preferred channels.

##### Maker offer

A maker offer is composed of:

 * The Arbitrating-Accordant blockchain identifier, e.g. BTC-XMR
 * The arbitrating blockchain asset amount
 * The accordant blockchain asset amount
 * The timelock durations used during the swap
 * The future maker swap role (Alice or Bob)

##### Maker public offer

A maker public offer is a maker offer plus daemon parameters, such as the onion service or other options detailing how to connect to the daemon.

The maker public offer must be easily serializable and in a friendly sharing format.

#### Taker

A taker receives a public offer, visualizes it and might accept it. If the taker wants to take the public offer he can try to connect to the maker and start the swap.

#### Results of negotiation phase

The negotiation is out of scope for this RFC but MUST result in:

 * Daemons connected to each other
 * Agreement on the following set of parameters
     * Blockchains used as an Arbitrating-Accordant asset pair, e.g. BTC-XMR
     * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
     * Transition from Maker-Taker roles into Alice-Bob roles
     * Time parameters t1 and t2, i.e. protocol timelock parameters

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/negociation-phase-mocks.png)

##### A. Maker and Taker role choice
The swap client is started in one of these two modes: (i) Maker or (ii) Taker.

##### B. Maker: create an offer
When started in maker mode, the client will ask the user a list of required parameters. When completed the client starts the daemon.

##### C. Taker: paste a public offer
When started in taker mode, the client prompt the user a public offer.

##### D. Maker: start and display the public offer
When daemon is ready, the client display the public offer and waits on incoming connection.

##### E. Taker: visualize a public offer
The pasted public offer is parsed and displayed to the user for verification and acceptance. If the user wants to take it, the daemon is started and connects to the daemon specified in the public offer.

#### CLI version
This section provides an incomplete example for the CLI version.

Maker starting his node:
```
$ swap-cli --make
> Pair: BTC-XMR
> BTC: 0.3
> XMR: 10
> Timelock 1: 4
> Timelock 2: 10
> Role ([A]lice,[B]ob): A

Exchange 0.3 BTC for 10 XMR? [y/N] y

Starting daemon in Maker mode...
Your public offer:
FxdquLnGbMMK9KsnW5P49hKb4vvKCjgjxJTd9tPE1PJr4f6LyLM

Waiting for incoming connection...
```

Taker visualizing a public offer and accepting the deal:
```
$ swap-cli --take --offer FxdquLnGbMMK9KsnW5P49hKb4vvKCjgjxJTd9tPE1PJr4f6LyLM
Public offer
> Pair: BTC-XMR
> BTC: 0.3
> XMR: 10
> Timelock 1: 4
> Timelock 2: 10
> Your role: Bob
> Counterparty role: Alice

Exchange 10 XMR for 0.3 BTC? [y/N] y

Connecting to counterparty daemon...
```

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

We describe a basic user experience with an atomic swap GUI client for Alice and Bob.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/gui-mocks.png)

##### 1. Initialization Step (1-2 in diagram)
Alice and Bob exchange the initialization parameters specified in [Inter-daemon protocol messages](/M0uYws_5S7K6k1j5l8b6qw?view)
and sign the multisig transactions
###### Messages exchanged: 
- Bob → Alice: [`send_initialization_parameters_bob`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_initialization_parameters_bob)
- Alice → Bob: [`send_initialization_parameters_alice`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_initialization_parameters_alice)
##### 2. Bitcoin Locking Step (3 in diagram)

After the parameters are exchanged, Bob acquires Alice's signatures for the refund path and locks the bitcoin. Bob signs the `BTX_buy` transaction and sends the transaction with his signature to Alice.
###### Messages exchanged: 
- Bob → Alice: [`send_bitcoin_transactions`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_bitcoin_transactions)
- Alice → Bob: [`send_bitcoin_transaction_signatures`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_bitcoin_transaction_signatures)
##### 3. Monero Locking Step (4 in diagram)
Once Alice has received sufficient confirmations for Bob's `BTX_lock` to feel safe and has received the signed `BTX_buy` transaction from Bob, Alice proceeds to lock her monero.
###### Messages exchanged: 
- Bob → Alice: [`send_bitcoin_buy_transaction`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_bitcoin_buy_transaction)
##### 4. Swap Step (5-6 in diagram)
Once Bob has received sufficient confirmations for Alice's `XTX_lock` to feel safe, Bob sends Alice the secret `s`, which Alice requires to execute the first branch of the `SWAPLOCK` script via `BTX_buy`. Alice then signs the `BTX_buy` transaction and publishes it, leaking her Monero key share. 
###### Messages exchanged: 
- Bob → Alice: [`send_bitcoin_buy_secret`](https://hackmd.io/M0uYws_5S7K6k1j5l8b6qw?view#send_bitcoin_buy_secret)

## Reputation asymmetry

Because of the protocol asymmetry, Alice always locks her coins later in the swap process, that implies she gets an option to buy without costs. One way to resolve this issue is to introduce a reputation system between participants, but this is hard in a decentralized setup.

The reputation asymmetry is not linked to the negotiation role assumed by Alice's daemon: If she's a Taker she can cancel for free on any prices and if she's a Maker she can propose any prices and cancel for free if someone tries to take it.