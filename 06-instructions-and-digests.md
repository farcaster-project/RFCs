<pre>
  State: draft
  Created: 2021-02-03
</pre>

# 06. Instructions & State Digests

## Overview

This RFC specifies the messages exchanged between the user's swap client and its own daemon.
As sketched elsewhere, the `client`→`daemon` route consists of (a) `instructions` from the client to daemon that control the state transitions of an ongoing swap, and (b) the `daemon`→`client` route consists of `state digests` sent to the client encoding the `daemon`'s swap state tailored specific to client's control and presentation functionality. The `client` must  present control choices to the end-user during the progression of the protocol execution.

```
                                      sk,pk       instructions     pk
                                     -----------  ------------> -------------
                                     | client  |                | daemon    |
                                     -----------  <-----------  -------------
                                                  state digests
```
*Fig 1. Sketch of interaction between a client and a daemon. Note only client has access to private keys (pk).*

`instructions` and `state digests` messages must follow the *'Type-Length-Value Format (TLV format)'* defines in *'BOLT #1: Base Protocol'* [[1](#references)] standard from the Lightning Network specifications.

## Table of Contents

  * [Security considerations](#security-considerations)
  * [Instructions](#instructions)
    * [The `alice_session_params` Instruction](#the-alice_session_params-instruction)
    * [The `bob_session_params` Instruction](#the-bob_session_params-instruction)
    * [The `cosigned_arbitrating_cancel` Instruction](#the-cosigned_arbitrating_cancel-instruction)
    * [The `signed_adapted_buy` Instruction](#the-signed_adapted_buy-instruction)
    * [The `fully_signed_buy` Instruction](#the-fully_signed_buy-instruction)
    * [The `signed_adapted_refund` Instruction](#the-signed_adapted_refund-instruction)
    * [The `fully_signed_refund` Instruction](#the-fully_signed_refund-instruction)
    * [The `signed_arbitrating_lock` Instruction](#the-signed_arbitrating_lock-instruction)
    * [The `abort` Instruction](#the-abort-instruction)
    * [The `next` Instruction](#the-next-instruction)
  * [State digests](#state-digests)
    * [The `state_digest` Message](#the-state_digest-message)
  * [References](#references)

## Security considerations

From a security perspective, an important distinction between the client and the daemon is that the daemon only knows public keys - private keys are the privy treasure of the client`(*)`. Nonetheless, the daemon MUST be viewed as a trusted component, since it exclusively verifies the correctness of the counterparty's data, controls the swap state, and can misreport progression of the swap to the client or mislead the client into invalid protocol states.

For instance, if the client is Bob who initially owns BTC in a swap, and the cancel path is invoked, if the client signs the `refund (e)` transaction and instructs the daemon to relay it, a malicious daemon could abstain from relaying it, resulting in a loss of funds for Bob, if he does not detect this error and submit the signed transaction via an alternate route before Alice can submit the `punish (f)` transaction to punish Bob `(**)`.

`*` *With the exception of all privates keys needed to read the blockchain state, e.g. the private view key when the accordant blockchain is Monero.*

`**` *For a better understanding of the transaction structure see [08. Transactions](./08-transactions.md).*

## Instructions

`instructions` must convey all required data a daemon needs to fulfill its mission, not only the keys or the transactions' signatures but also the user's commands for continuing the swap or cancelling it.

We define three categories of content found in `instructions`:

1. Results of cryptographic operation
   - Signatures (partial and finalized)
   - Keys (public keys and exceptional private 'view' keys)
   - Off-chain multi signature protocol messages (e.g., musig2)
   - Zero knowledge proofs requires by the above protocols (e.g., cross-group discreet log equality)
2. Transactions; following *PSBT standard* & *BIP 174* [[2,3]](#references)
3. Control flow operations
   - Accepting a step in the swap process
   - User canceling the swap

#### Control flow operations

We illustrate the effect client's control flow operations exert over a daemon, and its feedback loop back to the client. Both client and daemon have the responsability to exchange valid `instruction` and `state digest` messages based on their respective state and user actions. Please see the trust assumptions at [security considerations](#security-considerations).

A protocol transition moves the protocol execution forward -- that is a step in the swap process. The set of states that fulfills the predicates for enabling a given transition must be met, in order to be able to carry out the step in the swap process.

Please find below a high-level summary of this interaction:

1. A valid client `instruction` message sent by the client to the daemon controls the daemon to fire a given enabled protocol transition. 
2. Daemon consumes client `instruction` control flow message and
3. Daemon fires transitions that are in one-to-one correspondence with client instructions (if predicate conditions met)
4. As a consequence of firing protocol transitions, daemon's internal swap state may be modified
5. If the swap state was modified, daemon must update client by sending a `state digest` message to the client informing it about the new swap state and thus what enabled protocol transitions client may next instruct to fire
6. Client then may fire any of the new enabled protocol transition and progress on the protocol execution (back to step 1)

### The `alice_session_params` Instruction

**Send by**: Alice clients

Provides the counter-party daemon with all the information required for the initialization step of a swap.

> This message has the same format as the protocol message 33703 `areveal`, we have to check if two messages are needed or if we need only once and use it in two different context.
> I bet there is missing data here that the daemon needs to know, which would justify two messages.

 1. type: ? (`alice_session_params`)
 2. data:
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `buy`] The buy `Ab` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `cancel`] The cancel `Ac` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `refund`] The refund `Ar` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `punish`] The punish `Ap` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `adaptor`] The `Ta` adaptor public key
    - [`u16`: `address_len`]
    - [`address_len * byte`: `address`] The destination Bitcoin address
    - [`u16`: `ed25519_scalar_len`]
    - [`ed25519_scalar_len * byte`: `view`] The `K_v^a` view private key
    - [`u16`: `ed25519_point_len`]
    - [`ed25519_point_len * byte`: `spend`] The `K_s^a` spend public key
    - [`u16`: `proof_len`]
    - [`proof * byte`: `proof`] The cross-group discrete logarithm zero-knowledge proof


### The `bob_session_params` Instruction

**Send by**: Bob clients

Provides the counter-party daemon with all the information required for the initialization step of a swap.

> Same remarks have previous inst.

 1. type: ? (`bob_session_params`)
 2. data:
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `buy`] The buy `Bb` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `cancel`] The cancel `Bc` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `refund`] The refund `Br` public key
    - [`u16`: `secp256k1_point_len`]
    - [`secp256k1_point_len * byte`: `adaptor`] The `Tb` adaptor public key
    - [`u16`: `address_len`]
    - [`address_len * byte`: `address`] The refund Bitcoin address
    - [`u16`: `ed25519_scalar_len`]
    - [`ed25519_scalar_len * byte`: `view`] The `K_v^b` view private key
    - [`u16`: `ed25519_point_len`]
    - [`ed25519_point_len * byte`: `spend`] The `K_s^b` spend public key
    - [`u16`: `proof_len`]
    - [`proof * byte`: `proof`] The cross-group discrete logarithm zero-knowledge proof


### The `cosigned_arbitrating_cancel` Instruction

**Send by**: Alice|Bob clients

Provides daemon with a signature on the unsigned `cancel (d)` transaction previously provided by the daemon via `state digest`.

 1. type: ? (`cosigned_arbitrating_cancel`)
 2. data:
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Ac|Bc` `cancel (d)` signature


### The `signed_adapted_buy` Instruction

**Send by**: Bob clients

Provides Bob's daemon with a signature on the unsigned `buy (c)` transaction previously provided by the daemon via `state digest`.

 1. type: ? (`signed_adapted_buy`)
 2. data:
    - [`u16`: `buy_adaptor_sig_len`]
    - [`buy_adaptor_sig_len * byte`: `buy_adaptor_sig`] The `Bb(Ta)` `buy (c)` adaptor signature


### The `fully_signed_buy` Instruction

**Send by**: Alice clients

Provides Alice's daemon with the two signatures on the unsigned `buy (c)` transaction previously provided by the daemon via `state digest`, ready to be broadcasted.

 1. type: ? (`fully_signed_buy`)
 2. data:
    - [`u16`: `buy_sig_len`]
    - [`buy_sig_len * byte`: `buy_sig`] The `Ab` `buy (c)` signature
    - [`u16`: `buy_adapted_sig_len`]
    - [`buy_adapted_sig_len * byte`: `buy_adapted_sig`] The decrypted `Bb(Ta)` `buy (c)` adaptor signature

### The `signed_adapted_refund` Instruction

**Send by**: Alice clients

Provides Alice's daemon with a signature on the unsigned `refund (e)` transaction previously provided by the daemon via `state digest`.

 1. type: ? (`signed_adapted_refund`)
 2. data:
    - [`u16`: `refund_adaptor_sig_len`]
    - [`refund_adaptor_sig_len * byte`: `refund_adaptor_sig`] The `Ar(Tb)` `refund (e)` adaptor signature


### The `fully_signed_refund` Instruction

**Send by**: Bob clients

Provides Bob's daemon with the two signatures on the unsigned `refund (e)` transaction previously provided by the daemon via `state digest`, ready to be broadcasted.

 1. type: ? (`fully_signed_refund`)
 2. data:
    - [`u16`: `refund_sig_len`]
    - [`refund_sig_len * byte`: `refund_sig`] The `Br` `refund (e)` signature
    - [`u16`: `refund_adapted_sig_len`]
    - [`refund_adapted_sig_len * byte`: `refund__adapted_sig`] The decrypted `Ar(Tb)` `refund (e)` adaptor signature


### The `signed_arbitrating_lock` Instruction

**Send by**: Bob clients

Provides Bob's daemon with the signature on the unsigned `lock (b)` transaction previously provided by the daemon via `state digest`, ready to be broadcasted with this signature.

 1. type: ? (`signed_arbitrating_lock`)
 2. data:
    - [`u16`: `lock_sig_len`]
    - [`lock_sig_len * byte`: `lock_sig`] The `Bf` `lock (b)` signature for unlocking the funding


### The `signed_arbitrating_punish` Instruction

**Send by**: Alice clients

Provides Alice's daemon with the signature on the unsigned `punish (f)` transaction previously provided by the daemon via `state digest`, ready to be broadcasted with this signature.

 1. type: ? (`signed_arbitrating_punish`)
 2. data:
    - [`u16`: `punish_sig_len`]
    - [`punish_sig_len * byte`: `punish_sig`] The `Ap` `punish (f)` signature for unlocking the cancel transaction UTXO


### The `abort` Instruction

**Send by**: Alice|Bob clients

Provides deamon the instruction to abort the swap, it is the daemon responsability to abort accordingly to the current state swap. By transmitting latter feedback via `state digest`, the client must be able to provide any missing signatures.

 1. type: ? (`abort`)
 2. data:
    - [`u16`: `abort_code`] OPTIONAL: A code conveying the reason of the abort


### The `next` Instruction

**Send by**: Alice|Bob clients

Provides deamon the instruction to follow the protocol swap, daemon can create locking steps during the protocol execution and require client to acknoledge the execution progression.

The `next_code` may be used when next require a choice by the client.

 1. type: ? (`next`)
 2. data:
    - [`u16`: `next_code`] OPTIONAL: A code conveying the type of execution progression


## State digests
State digests are messages sent by the daemon to the client to relay all information needed to construct the client interface and update the swap state. These messages both allow the client to display the correct actions a user can perform and create the necessary instructions consumed daemon to continue the swap protocol.

**Format**:

- Current marking of petri net representation of swap
- Data required for firing a given transition

### The `state_digest` Message
Provides the client with the current swap state digest. By applying the fired transitions to the current petri net of the client, the client can infer which transitions are available to fire.

 1. type: 33790 (`state_digest`)
 2. data:
    - [`u16`: `fired_transitions_len`]
    - [`fired_transitions_len * type`: `fired_transitions`] Vector of # of firings per transition since last state digest
    - [`sha256`: `marking_hash`] Hash of the current marking so that client can verify it applied the transitions correctly
    - [`u16`: `fireable_transition_data_len`]
    - [`fireable_transition_data_len * type`: `fireable_transition_data`] Map of fireable transitions to the data that the client requires to produce the authentication the daemon requires to fire it, such as transaction signatures

## References

 * [[1] BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format)
 * [[2] PSBT standard](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)
 * [[3] BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
