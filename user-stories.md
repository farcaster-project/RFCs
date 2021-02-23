
[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)

<pre>
  State: draft
  Created: 2020-11-11
</pre>

# User Stories

## Overview

This RFC describe the roles and phases of a swap and presents simple user stories, mockups and basic cli example.

## Table of Contents

[TOC]

## Phases

We distinguish between two phases: (1) negotiation and (2) swap phases. The negotiation implies taker and maker roles, the swap phase implies Alice and Bob roles.

### Negotiation phase

The negotiation phase can be done on a forum, with an OTC, within an DEX, etc. This RFC defines the interface between the two phases: negotiation and swap, and propose a minimalistic negociation protocol.

It is worth mentionning that the negociation phase proposed in this RFC does not contain any negociation mechanism. It would be more accuratly described as a *connect and accept* mechanism. Thus this mechanism can be extended and external matching engines could implement a negociation protocol. These negociation protocol could then end up with the described connect and accept protocol.

#### Maker

The maker starts her node in maker mode and registers all the parameters an offer requires. The daemon creates an onion service and waits for an incoming connection. When ready, the daemon prints the 'public offer'. The public offer contains all the parameters a taker needs to connect. The maker can then distribute the 'public offer' over her preferred channels.

##### Maker offer

A maker offer is composed of:

 * The Arbitrating/Accordant blockchain identifier, e.g. BTC-XMR; *Must indentify: blockchain chain and asset traded. E.g. bitcoin on mainnet or bitcoin on testnet. This can be done through a `chain_hashes` parameter.*
 * The arbitrating blockchain asset amount
 * The accordant blockchain asset amount
 * The timelock durations used during the swap
 * The fee calculation strategy; *This might be static, within a range, or define as a function*
 * The future maker swap role (Alice or Bob); *Taker role is derived from this as the protocol always have one Alice and one Bob*

##### Maker's public offer

A maker public offer is an extended maker offer with daemon's network parameters, such as the onion service or other options detailing how to connect to the daemon. The public offer must specify the node's long-term identifier and must contain a valid signature.

The maker public offer must be as user friedly as possible, as it is the responsability to the user to share this public offer with potential counter-party via maker's prefered communication channels.

#### Taking a public offer

A taker receives a public offer, parses and visualizes it and might accept it. If the taker wants to take the public offer he can try to connect to the maker and start the swap. As this protocol doesn't ensure that the public offer is still live nor already taken, this action can fail.

#### Results of negotiation phase

At the end of the negotiation phase, participants must result with:

 * Daemons connected to each other
 * Validated set of parameters containing:
     * Blockchains used as an Arbitrating-Accordant asset pair, e.g. BTC-XMR
     * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
     * Transition from Maker-Taker roles into Alice-Bob roles
     * Time parameters t1 and t2, i.e. protocol timelock parameters
     * Fee strategy for arbitrating transactions

#### GUI Mockups

We present a simple user interface for starting with the role of a maker or a taker and the steps that must follow each choice.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/negociation-phase-mocks.png)

##### A. Maker and Taker role choice
The swap client is started in one of these two modes: Maker or Taker.

##### Maker
**B. Create an offer**
When started in maker mode, the client will ask the user a list of required parameters (see "Maker offer" below). When completed the client starts the daemon.

**D. Start and display the public offer**
When daemon is ready, the client display the public offer and waits on incoming connection.

##### Taker
**C. Paste a public offer**
When started in taker mode, the client prompt the user a public offer.

**E. Visualize a public offer**
The pasted public offer is parsed and displayed to the user for verification and acceptance. If the user wants to take it, the daemon is started and connects to the daemon specified in the public offer.

#### CLI version
This section presents an example of a CLI swap client. This is provided for educational purposes, the swap CLI client may look different.

**Maker** starting his node:
```
$ swap-cli --make
> Pair: BTC-XMR
> BTC: 0.3
> XMR: 10
> Timelock 1: 4
> Timelock 2: 10
> Fee strategy ([F]ixe, [R]ange, [D]ynamique): R
> Fee range (sat/vB): 10-40
> Role ([A]lice,[B]ob): A

You chose Alice:
> Bitcoin destination address: bc1qndk902ka3266wzta9cnl4fgfcmhy7xqrdh26ka

Exchange 0.3 BTC for 10 XMR? [y/N] y

Starting daemon in Maker mode...

Your public offer:
FxdquLnGbMMK9KsnW5P49hKb4vvKCjgjxJTd9tPE1PJr4f6LyLM

Waiting for incoming connection...
```

**Taker** visualizing a public offer and accepting the deal:
```
$ swap-cli --take --offer FxdquLnGbMMK9KsnW5P49hKb4vvKCjgjxJTd9tPE1PJr4f6LyLM
Public offer
: Pair: BTC-XMR
: BTC: 0.3
: XMR: 10
: Timelock 1: 4
: Timelock 2: 10
: Fee range (sat/vB): 10-40
: Your role: Bob
: Counterparty role: Alice

Exchange 10 XMR for 0.3 BTC? [y/N] y

You are Bob:
> Bitcoin refund address: bc1qkj370d9hc3seqauu9sujm96aqfw5ml9a46ejfa

Connecting to counterparty daemon...
```

### Swap phase

The swap phase starts for each roles with a common set of parameters:

 * Blockchains used as an Arbitrating-Accordant asset pair, e.g. BTC-XMR
 * Amount exchanged of each assets, e.g. 200 XMR and 1.3 BTC
 * Time parameters t1 and t2, i.e. protocol timelock parameters
 * The fee strategy and its chosen parameters
 * The swap role played by the daemon

plus for Alice's role:

 * The destination Bitcoin address

and for Bob's role:

 * The refund Bitcoin address

#### Steps

We describe the high level view of the swap phase with four steps:

 1. Initialization step
 2. Bitcoin locking step
 3. Monero locking step
 4. Swap step

We describe a basic user experience with an atomic swap GUI client for Alice and Bob. This is provided for educational purposes and to give an idea to the reader, the swap GUI client may look different.

![](https://raw.githubusercontent.com/farcaster-project/RFCs/hackmd/images/gui-mocks.png)

##### 1. Initialization Step (1 in diagram)
Alice and Bob start the pre-initialization. They exchange and verify parameters specified in [Protocol messages](/M0uYws_5S7K6k1j5l8b6qw?view) RFC. If the validation successfully terminates, the client moves to the next step.

###### Messages exchanged:

*First round, commit to values*

> Messages can arrive in any order

- Alice → Bob: [`commit_alice_session_params`]()
- Bob → Alice: [`commit_bob_session_params`]()

*Second round, reveal values*

> Messages can arrive in any order

- Alice → Bob: [`reveal_alice_session_params`]()
- Bob → Alice: [`reveal_bob_session_params`]()

##### 2. Bitcoin Locking Step (2-3 in diagram)
After the parameters are exchanged and validated, Bob ask the user for funding. Uppon funds reception Bob creates the transactions, signs the cancel path and sends them to Aline with `core_arbitrating_setup` protocol message. He acquires Alice's signatures for the cancel path. The bitcoin are locked when Bob is able to trigger the cancel path and refund the assets, i.e. after reception of `refund_procedure_signatures` protocol message.

###### Messages exchanged:

- Bob → Alice: [`core_arbitrating_setup`]()
- Alice → Bob: [`refund_procedure_signatures`]()

##### 3. Monero Locking Step (4 in diagram)
Once Alice has received sufficient confirmations for Bob's `lock (b)` transaction to feel safe, Alice proceeds to lock her monero with the Monero `lock (a)` transaction.

##### 4. Swap Step (5-6 in diagram)
Once Bob has received sufficient confirmations for the Monero `lock (a)` transaction to feel safe, Bob sends Alice the `buy (c)` encrypted signature, which Alice requires to execute the first branch of the `lock (b)` transaction output script via the `buy (c)` transaction.

Alice then signs the `buy (c)` transaction to complete it and publishes it, leaking her Monero key share and finalizing her swap at the same time. Bob sees the `buy (c)` transaction in the mempool, extract the Monero key share and display it to the user.

###### Message exchanged:

- Bob → Alice: [`buy_procedure_signature`]()
