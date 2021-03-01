<pre>
  State: draft
  Created: 2021-02-03
</pre>

# 06. Instructions and Proposals

## Overview

This RFC specifies the messages exchanged between the user's swap client and its own daemon.
As sketched below, the `client`→`daemon` route consists of (a) `instructions` from the client to daemon that control the state transitions of an ongoing swap, and (b) the `daemon`→`client` route consists of `proposals` sent to the client encoding the `daemon`'s swap state and proposing available instructions for client's control and presentation functionality. `proposal` messages include the backbone of instruction messages `proposals` are described in [09. Swap state](./09-swap-state.md) The `client` must  present control choices to the end-user during the progression of the protocol execution.

```
                                      sk,pk       instructions     pk
                                     -----------  ------------> -------------
                                     | client  |                | daemon    |
                                     -----------  <-----------  -------------
                                                    proposals
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
  * [Proposals](#proposals)
    * [The `cosign_arbitrating_cancel` Proposal](#the-cosign_arbitrating_cancel-proposal)
    * [The `sign_adaptor_buy` Proposal](#the-sign_adaptor_buy-proposal)
    * [The `fully_sign_buy` Proposal](#the-fully_sign_buy-proposal)
    * [The `sign_adaptor_refund` Proposal](#the-sign_adaptor_refund-proposal)
    * [The `fully_sign_refund` Proposal](#the-fully_sign_refund-proposal)
    * [The `sign_arbitrating_lock` Proposal](#the-sign_arbitrating_lock-proposal)
    * [The `sign_arbitrating_punish` Proposal](#the-sign_arbitrating_punish-proposal)
  * [References](#references)


## Security considerations

From a security perspective, an important distinction between the client and the daemon is that the daemon only knows public keys - private keys are the privy treasure of the client`(*)`. Nonetheless, the daemon MUST be viewed as a trusted component, since it exclusively verifies the correctness of the counterparty's data, controls the swap state, and can misreport progression of the swap to the client or mislead the client into invalid protocol states.

For instance, if the client is Bob who initially owns BTC in a swap, and the cancel path is invoked, if the client signs the `refund (e)` transaction and instructs the daemon to relay it, a malicious daemon could abstain from relaying it, resulting in a loss of funds for Bob, if he does not detect this error and submit the signed transaction via an alternate route before Alice can submit the `punish (f)` transaction to punish Bob `(**)`.

`*` *With the exception of all privates keys needed to read the blockchain state, e.g. the private view key when the accordant blockchain is Monero.*

`**` *For a better understanding of the transaction structure see [08. Transactions](./08-transactions.md).*

## Instructions: Low level

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

Provides daemon with a signature on the unsigned `cancel (d)` transaction previously provided by the daemon via `cosign_arbitrating_cancel` Proposal.

 1. type: ? (`cosigned_arbitrating_cancel`)
 2. data:
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Ac|Bc` `cancel (d)` signature


### The `signed_adaptor_buy` Instruction

**Send by**: Bob clients

Provides Bob's daemon with an adaptor signature for the unsigned `buy (c)` transaction. In response to daemon's `sign_adaptor_buy` Proposal.

 1. type: ? (`signed_adaptor_buy`)
 2. data:
    - [`u16`: `buy_adaptor_sig_len`]
    - [`buy_adaptor_sig_len * byte`: `buy_adaptor_sig`] The `Bb(Ta)` `buy (c)` adaptor signature

### The `fully_signed_buy` Instruction

**Send by**: Alice clients

Provides Alice's daemon with the two signatures on the unsigned `buy (c)` transaction previously provided by the daemon via `fully_sign_buy` Proposal, ready to be broadcasted.

 1. type: ? (`fully_signed_buy`)
 2. data:
    - [`u16`: `buy_sig_len`]
    - [`buy_sig_len * byte`: `buy_sig`] The `Ab` `buy (c)` signature
    - [`u16`: `buy_adapted_sig_len`]
    - [`buy_adapted_sig_len * byte`: `buy_adapted_sig`] The decrypted `Bb(Ta)` `buy (c)` adaptor signature

### The `signed_adaptor_refund` Instruction

**Send by**: Alice clients

Provides Alice's daemon with a signature on the unsigned `refund (e)` transaction previously provided by the daemon via `sign_adaptor_refund` Proposal.

 1. type: ? (`signed_adaptor_refund`)
 2. data:
    - [`u16`: `refund_adaptor_sig_len`]
    - [`refund_adaptor_sig_len * byte`: `refund_adaptor_sig`] The `Ar(Tb)` `refund (e)` adaptor signature


### The `fully_signed_refund` Instruction

**Send by**: Bob clients

Provides Bob's daemon with the two signatures on the unsigned `refund (e)` transaction previously provided by the daemon via `fully_sign_refund` Proposal, ready to be broadcasted.

 1. type: ? (`fully_signed_refund`)
 2. data:
    - [`u16`: `refund_sig_len`]
    - [`refund_sig_len * byte`: `refund_sig`] The `Br` `refund (e)` signature
    - [`u16`: `refund_adapted_sig_len`]
    - [`refund_adapted_sig_len * byte`: `refund_adapted_sig`] The decrypted `Ar(Tb)` `refund (e)` signature


### The `signed_arbitrating_lock` Instruction

**Send by**: Bob clients

Provides Bob's daemon with the signature on the unsigned `lock (b)` transaction previously provided by the daemon via `sign_arbitrating_lock` Proposal, ready to be broadcasted with this signature.

 1. type: ? (`signed_arbitrating_lock`)
 2. data:
    - [`u16`: `lock_sig_len`]
    - [`lock_sig_len * byte`: `lock_sig`] The `Bf` `lock (b)` signature for unlocking the funding


### The `signed_arbitrating_punish` Instruction

**Send by**: Alice clients

Provides Alice's daemon with the signature on the unsigned `punish (f)` transaction previously provided by the daemon via `sign_arbitrating_punish` Proposal, ready to be broadcasted with this signature.

 1. type: ? (`signed_arbitrating_punish`)
 2. data:
    - [`u16`: `punish_sig_len`]
    - [`punish_sig_len * byte`: `punish_sig`] The `Ap` `punish (f)` signature for unlocking the cancel transaction UTXO

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


## Proposals

Proposals are messages sent from Daemon to Client providing data Client needs to proceed in the protocol execution. They propose transactions, presents counterparty signatures and adaptor signatures to Client. With these data, Client is able to create valid, fully signed transactions.

### The `cosign_arbitrating_cancel` Proposal
**Send by**: Alice|Bob daemon
- data:
    - [`u16`: `cancel_tx_len`]
    - [`cancel_tx_len * byte`:`cancel (d)`] The `cancel (d)` transaction

### The `sign_adaptor_buy` Proposal
**Send by**: Bob daemon
- data:
    - [`buy (c)`] buy transaction

### The `fully_sign_buy` Proposal
**Send by**: Alice daemon
- data:
    - [`u16`: `buy_adaptor_sig_len`]
    - [`buy_adaptor_sig_len * byte`: `buy_adaptor_sig`] The `Bb(Ta)` `buy (c)` adaptor signature

### The `sign_adaptor_refund` Proposal
**Send by**: Alice daemon
data:
    - [`u16`: `refund_tx_len`]
    - [`refund_tx_len * byte`:`refund (e)`] The refund transaction

### The `fully_sign_refund` Proposal
**Send by**: Bob daemon
- data:
    - [`u16`: `refund_tx_len`]
    - [`refund_tx_len * byte`:`refund (e)`] The `refund (e)` transaction
    - [`u16`: `refund_adaptor_sig_len`]
    - [`refund_adaptor_sig_len * byte`: `refund_adaptor_sig`] The `Ar(Tb)` `refund (e)` adaptor signature

### The `sign_arbitrating_lock` Proposal
**Send by**: Bob daemon
- data:
    - [`u16`: `lock_tx_len`]
    - [`lock_tx_len * byte`:`lock (b)`] The `lock` transaction

### The `sign_arbitrating_punish` Proposal
**Send by**: Alice daemon
- data
    - [`u16`: `punish_tx_len`]
    - [`punish_tx_len * byte`:`punish (f)`] The `punish (f)` transaction


## References
 * [[1] BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format)
 * [[2] PSBT standard](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md)
 * [[3] BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)

