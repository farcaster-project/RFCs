<pre>
  State: draft
  Created: 2021-02-03
</pre>

# 06. Instructions

## Overview

This RFC specifies the messages exchanged between the user's swap client and its own daemon.
As sketched below, the `client`→`daemon` and `daemon`→`client` routes consists of `instructions`. They control the state transitions of an ongoing swap. The `client` must  present control choices to the end-user during the progression of the protocol execution.

```
                                      sk,pk                        pk
                                     -----------  ------------> -------------
                                     | client  |  instructions  | daemon    |
                                     -----------  <-----------  -------------

```
*Fig 1. Sketch of interaction between a client and a daemon. Note only client has access to private keys (pk).*

`instructions` messages must follow the *'Type-Length-Value Format (TLV format)'* defines in *'BOLT #1: Base Protocol'* [[1](#references)] standard from the Lightning Network specifications.

## Table of Contents
  * [Security considerations](#security-considerations)
  * [Instructions: Low level](#instructions-low-level)
    * [The `alice_session_params` Instruction](#the-alice_session_params-instruction)
    * [The `bob_session_params` Instruction](#the-bob_session_params-instruction)
    * [The `cosigned_arbitrating_cancel` Instruction](#the-cosigned_arbitrating_cancel-instruction)
    * [The `signed_adaptor_buy` Instruction](#the-signed_adaptor_buy-instruction)
    * [The `fully_signed_buy` Instruction](#the-fully_signed_buy-instruction)
    * [The `signed_adaptor_refund` Instruction](#the-signed_adaptor_refund-instruction)
    * [The `fully_signed_refund` Instruction](#the-fully_signed_refund-instruction)
    * [The `signed_arbitrating_lock` Instruction](#the-signed_arbitrating_lock-instruction)
    * [The `signed_arbitrating_punish` Instruction](#the-signed_arbitrating_punish-instruction)
  * [Instructions: High level, Control flow messages](#instructions-high-level-control-flow-messages)
    * [The `abort` Instruction](#the-abort-instruction)
      * [Daemon's  response to `abort` Instruction](#daemons--response-to-abort-instruction)
    * [The `next` Instruction](#the-next-instruction)
  * [References](#references)


## Security considerations

From a security perspective, an important distinction between the client and the daemon is that the daemon only knows public keys - private keys are the privy treasure of the client`(*)`. Nonetheless, the daemon MUST be viewed as a trusted component, since it exclusively verifies the correctness of the counterparty's data, controls the swap state, and can misreport progression of the swap to the client or mislead the client into invalid protocol states.

For instance, if the client is Bob who initially owns BTC in a swap, and the cancel path is invoked, if the client signs the `refund (e)` transaction and instructs the daemon to relay it, a malicious daemon could abstain from relaying it, resulting in a loss of funds for Bob, if he does not detect this error and submit the signed transaction via an alternate route before Alice can submit the `punish (f)` transaction to punish Bob `(**)`.

`*` *With the exception of all privates keys needed to read the blockchain state, e.g. the private view key when the accordant blockchain is Monero.*

`**` *For a better understanding of the transaction structure see [08. Transactions](./08-transactions.md).*

## Instructions: Low level

`instructions` must convey all required data a daemon or a client needs to fulfill its mission, not only the keys or the transactions' signatures but also the user's commands for continuing the swap or cancelling it.

We define three categories of content found in `instructions`:

1. Results of cryptographic operation
   - Signatures (partial and finalized)
   - Keys (public keys and exceptional private 'view' keys)
   - Off-chain multi signature protocol messages (e.g., musig2)
   - Zero knowledge proofs requires by the above protocols (e.g., cross-group discreet log equality)
2. Transactions; following *PSBT standard* & *BIP 174* [[2,3]](#references)
3. Control flow operations
   - Accepting a step in the swap process
   - User or protocol canceling the swap

### The `transation` Instruction

The `transaction` instruction is used to convey a transaction between clients and daemons. The transaction is transmitted within the `tx_value` field in its serialized format.

 1. type: ? (`transation`)
 2. data:
    - [`u16`: `tx_id`] The identifier of the transaction
    - [`u16`: `tx_value_len`]
    - [`tx_value_len * byte`: `tx_value`] The serialized value of the transaction

The type of the transaction is derived from the `tx_id`:

 1. `tx_id`:
    - `0x01`: `funding (a)` arbitrating transaction
    - `0x02`: `lock (b)` arbitrating transaction
    - `0x03`: `buy (c)` arbitrating transaction
    - `0x04`: `cancel (d)` arbitrating transaction
    - `0x05`: `refund (e)` arbitrating transaction
    - `0x06`: `punish (f)` arbitrating transaction

### The `key` Instruction

The `key` instruction is used to convey keys between clients and daemons. The key is transmitted within the `key_value` field in its serialized format and is identified by the `key_id`.

 1. type: ? (`key`)
 2. data:
    - [`u16`: `key_id`] The identifier of the key
    - [`u16`: `key_value_len`]
    - [`key_value_len * byte`: `key_value`] The serialized value of the key

The type of the key is derived from the `key_id`:

 1. `key_id`:
    - `0x01`: `Ab` Alice buy key
    - `0x02`: `Ac` Alice cancel key
    - `0x03`: `Ar` Alice refund key
    - `0x04`: `Ap` Alice punish key
    - `0x05`: `Ta` Alice adaptor key
    - `0x06`: `K_s^a` Alice spend key
    - `0x07`: `K_v^a` Alice private view key
    - `0x08`: `Bf` Bob fund key
    - `0x09`: `Bb` Bob buy key
    - `0x0a`: `Bc` Bob cancel key
    - `0x0b`: `Br` Bob refund key
    - `0x0c`: `Tb` Bob adaptor key
    - `0x0d`: `K_s^b` Bob spend key
    - `0x0e`: `K_v^b` Bob private view key

### The `signature` Instruction

The `signature` instruction is used to convey signatures between clients and daemons. When this instruction comes from a client it is usually a signature freshly generated or adapted by the client, when the instruction is emitted by the daemon to the client it is usually a adaptor signature to be adapted by the client.

 1. type: ? (`signature`)
 2. data:
    - [`u16`: `tx_id`] The identifier of the related transaction, see `tx_id` definition
    - [`u16`: `role`] The swap role that emitted the signature
    - [`u16`: `sig_value_len`]
    - [`sig_value_len * byte`: `sig_value`] The serialized value of the signature, the signature can be a adaptor, adapted, or regular signature

The swap role:

 1. `role`:
    - `0x01`: Alice
    - `0x02`: Bob

### The `proof` Instruction

The `proof` instruction is used by clients to provides cryptographic proofs needed to secure the protocol.

 1. type: ? (`proof`)
 2. data:
    - [`u16`: `proof_id`] The identifier of the proof
    - [`u16`: `proof_value_len`]
    - [`proof_value_len * byte`: `proof_value`] The serialized value of the proof

The proof type is derived from the `proof_id`:

 1. `proof_id`:
    - `0x01`: Cross-group discrete logarthim proof

### The `parameter` Instruction

The `parameter` instruction is used to convey parameters between clients and daemons such as: addresses, timelocks, fee strategies, etc. They are mostly used by clients to instruct daemons about user parameters and offer parameters.

 1. type: ? (`parameter`)
 2. data:
    - [`u16`: `param_id`] The identifier of the parameter
    - [`u16`: `param_value_len`]
    - [`param_value_len * byte`: `param_value`] The serialized value of the parameter

The type of the parameter is derived from the `param_id`:

 1. `param_id`:
    - `0x01`: Alice destination address
    - `0x02`: Bob refund address
    - `0x03`: Cancel timelock
    - `0x04`: Punish timelock
    - `0x05`: Fee strategy

## Instructions: High level, Control flow messages

The low level Instructions above behaves as if the Daemon controls the Client. The High level, control flow messages, triggers the Daemon to initiate the apparent control of the client.

We illustrate the effect client's control flow operations exert over a daemon, and its feedback loop back to the client. Both client and daemon have the responsibility to exchange valid `instruction` and `Proposal` messages based on their respective state and user actions. Please see the trust assumptions at [security considerations](#security-considerations).

A protocol transition moves the protocol execution forward, that is a step in the swap process. The set of states that fulfills the predicates for enabling a given transition must be selected, in order to be able to carry out the step in the swap process.

Please find below a high-level summary of this interaction:

 1. A valid client `instruction` message sent by the client to the daemon controls the daemon to fire a given enabled protocol transition.
 2. Daemon consumes client `instruction` control flow message and
 3. Daemon fires transitions that are in one-to-one correspondence with client instructions (if predicate conditions met)
 4. As a consequence of firing protocol transitions, daemon's internal swap state may be modified
 5. If the swap state was modified, daemon must send client `Proposal` messages providing client with the data for next user actions if any new actions available. When applicable, Daemon must as well spawn Syncer tasks.
 6. Client then may give new instructions and progress on the protocol execution (back to step 1)

### The `abort` Instruction

**Send by**: Alice|Bob clients

Provides daemon the instruction to abort the swap, it is the daemon responsability to abort accordingly to the current swap-state. Upon daemon request, via `proposal`, the client must be able to provide any missing signatures.

 1. type: ? (`abort`)
 2. data:
    - [`u16`: `abort_code`] OPTIONAL: A code conveying the reason of the abort

#### Daemon's  response to `abort` Instruction

**Send by**: Bob and Alice daemon

`abort` Instruction MAY trigger Tasks and Proposals, and their downstream effects, depending on who called it and the current swap-state, such as:
    - Bob or Alice: `publish_tx cancel` | `watch_tx cancel`
    - Bob: `fully_sign_refund`
    - Bob: `publish_tx refund` & `watch_tx refund`
    - Alice: `sign_arbitrating_punish`
    - Alice: `publish_punish`

### The `next` Instruction

**Send by**: Alice|Bob clients

Provides daemon the instruction to follow the swap protocol, daemon can create locking steps during the protocol execution and require client to acknowledge the execution progression.

The `next_code` may be used when next require a choice by the client.

 1. type: ? (`next`)
 2. data:
    - [`u16`: `next_code`] OPTIONAL: A code conveying the type of execution progression

## Instruction bundles

Instruction decribe above are succint and are used to convey atomic chunk of data between clients and daemons. We also present here the bundles used during the different steps of the swap by both Alice and Bob. A bundle is a aggregate of 1 or more atomic `instruction` with a particular meaning. Meanings of each bundle is described with the list of content that should be received as atomic instructions.

### The `alice_session_params` Bundle

**Send by**: Alice clients|Bob daemon

Provides the (counter-party) daemon with all the information required for the initialization step of a swap.

 1. data:
    - The buy `Ab` public key
    - The cancel `Ac` public key
    - The refund `Ar` public key
    - The punish `Ap` public key
    - The `Ta` adaptor public key
    - The destination Bitcoin address
    - The `K_v^a` view private key
    - The `K_s^a` spend public key
    - The cross-group discrete logarithm zero-knowledge proof
    - The `cancel` and `punish` timelocks
    - The fee strategy


### The `bob_session_params` Bundle

**Send by**: Bob clients|Alice daemon

Provides the (counter-party) daemon with all the information required for the initialization step of a swap.

 1. data:
    - The buy `Bb` public key
    - The cancel `Bc` public key
    - The refund `Br` public key
    - The `Tb` adaptor public key
    - The refund Bitcoin address
    - The `K_v^b` view private key
    - The `K_s^b` spend public key
    - The cross-group discrete logarithm zero-knowledge proof
    - The `cancel` and `punish` timelocks
    - The fee strategy


### The `cosigned_arbitrating_cancel` Bundle

**Send by**: Alice|Bob clients

Provides daemon with a signature on the unsigned `cancel (d)` transaction.

 1. data:
    - The `Ac|Bc` `cancel (d)` signature


### The `signed_adaptor_buy` Bundle

**Send by**: Bob clients|Alice daemon

Provides Bob's daemon or Alice's client with an adaptor signature for the unsigned `buy (c)` transaction.

 1. data:
    - The `Bb(Ta)` `buy (c)` adaptor signature


### The `fully_signed_buy` Bundle

**Send by**: Alice clients|Bob daemon

Provides Alice's daemon or Bob's clients with the two signatures on the unsigned `buy (c)` transaction.

 1. data:
    - The `Ab` `buy (c)` signature
    - The adapted `Bb(Ta)` `buy (c)` adaptor signature


### The `signed_adaptor_refund` Bundle

**Send by**: Alice clients|Bob daemon

Provides Alice's daemon or Bob's clients with a signature on the unsigned `refund (e)` transaction.

 1. data:
    - The `Ar(Tb)` `refund (e)` adaptor signature


### The `fully_signed_refund` Bundle

**Send by**: Bob clients|Alice daemon

Provides Bob's daemon or Alice's clients with the two signatures on the unsigned `refund (e)` transaction.

 1. data:
    - The `Br` `refund (e)` signature
    - The adapted `Ar(Tb)` `refund (e)` adaptor signature


### The `signed_arbitrating_lock` Bundle

**Send by**: Bob clients

Provides Bob's daemon with the signature on the unsigned `lock (b)` transaction.

 1. data:
    - The `Bf` `lock (b)` signature for unlocking the funding


### The `signed_arbitrating_punish` Bundle

**Send by**: Alice clients

Provides Alice's daemon with the signature on the unsigned `punish (f)` transaction.

 1. data:
    - The `Ap` `punish (f)` signature for unlocking the cancel transaction UTXO

## References
 * [[1] BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format)
 * [[2] PSBT standard](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)
 * [[3] BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)

