<pre>
  State: draft
  Created: 2021-01-20
</pre>

# 04. Protocol Messages

## Overview

This RFC specifies the inter-daemon communication messages used during the swap phase, see [02. User Stories](./02-user-stories.md#1-initialization-step-1-in-the-diagram). Protocol messages are designed based on the protocol execution specified in [[1] Bitcoin-Monero Cross-chain Atomic Swap](#references) but generalized to generic arbitrating and accordant blockchains.

## Table of Contents

  * [Inter-daemon communication](#inter-daemon-communication)
  * [Messages](#messages)
    * [The `commit_alice_session_params` Message](#the-commit_alice_session_params-message)
    * [The `commit_bob_session_params` Message](#the-commit_bob_session_params-message)
    * [The `reveal_alice_session_params` Message](#the-reveal_alice_session_params-message)
    * [The `reveal_bob_session_params` Message](#the-reveal_bob_session_params-message)
    * [The `core_arbitrating_setup` Message](#the-core_arbitrating_setup-message)
    * [The `refund_procedure_signatures` Message](#the-refund_procedure_signatures-message)
    * [The `buy_procedure_signature` Message](#the-buy_procedure_signature-message)
    * [The `abort` Message](#the-abort-message)
  * [References](#references)

## Inter-daemon communication

The inter-daemon communication must follow [[2] BOLT #8: Encrypted and Authenticated Transport](#references) standard from the Lightning Network specifications.

All communications between nodes are encrypted to provide confidentiality for all transcripts between nodes and are authenticated to avoid malicious interference. Each node has a known connection identifier that is a public key on Bitcoin's `secp256k1` curve. This connection public key is used to establish an encrypted and authenticated connection with peers.

When starting in maker mode, a daemon will start a listening service. The listening service uses the connection public key to decrypt and authenticate incoming connections. The listening service may create an onion service to hide the node location or may listen on an IPv4/IPv6 address directly.

Node address and connection public key are registered into the public offer, which must be signed before been shared. Other daemons can then connect and create an encrypted and authenticated channel. It is worth noting that connecting peers are not authenticated by the listening node, anyone who knows the address can try to connect. For more detail on the public offer see [02. User Stories](./02-user-stories.md).

## Messages

The inter-daemon messages must follow the *Type-Length-Value Format* defined in [[3] BOLT #1: Base Protocol](#references) standard from the Lightning Network specifications.

The following convenience types extend 'BOLT #1' fundamental types:

 * `swap_id`: a 32-byte swap unique identifier

Cryptographic and transaction types are assumed to be generic, so e.g. different public key types can be used, i.e. multiple arbitrating blockchains with multiple cryptographic primitives can be supported later. These types such as `signature` and `adaptor_signature` are defined in [07. Cryptographic Setup](./07-cryptographic-setup.md) and may vary depending on the cryptography used on the arbitrating blockchain.

### The `commit_alice_session_params` Message

**Sent by**: Alice

`commit_alice_session_params` forces Alice to commit to the result of her cryptographic setup before receiving Bob's setup. This is done to remove adaptive behavior.

If not needed for the accordant blockchain the view key commitment must be set to a null hash.

 1. type: 33701 (`commit_alice_session_params`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`sha256`: `buy`] Commitment to `Ab`, an arbitrating curve point
    - [`sha256`: `cancel`] Commitment to `Ac`, an arbitrating curve point
    - [`sha256`: `refund`] Commitment to `Ar`, an arbitrating curve point
    - [`sha256`: `punish`] Commitment to `Ap`, an arbitrating curve point
    - [`sha256`: `adaptor`] Commitment to `Ta`, an arbitrating curve point
    - [`sha256`: `view`] Commitment to `k_v^a`, an accordant scalar
    - [`sha256`: `spend`] Commitment to `K_s^a`, an accordant curve point

### The `commit_bob_session_params` Message

**Sent by**: Bob

`commit_bob_session_params` forces Bob to commit to the result of his cryptographic setup before receiving Alice's setup. This is done to remove adaptive behavior.

If not needed for the accordant blockchain the view key commitment must be set to a null hash.

 1. type: 33702 (`commit_bob_session_params`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`sha256`: `buy`] Commitment to `Bb`, an arbitrating curve point
    - [`sha256`: `cancel`] Commitment to `Bc`, an arbitrating curve point
    - [`sha256`: `refund`] Commitment to `Br`, an arbitrating curve point
    - [`sha256`: `adaptor`] Commitment to `Tb`, an arbitrating curve point
    - [`sha256`: `view`] Commitment to `k_v^b`, an accordant scalar
    - [`sha256`: `spend`] Commitment to `K_s^b`, an accordant curve point

### The `reveal_alice_session_params` Message

**Sent by**: Alice

`reveal_alice_session_params` reveals the parameters commited by the `commit_alice_session_params` message.

If not needed for the accordant blockchain the view key value must be set to `0x0`.

If not needed for the pair of Arbitrating-Accordant blockchain the proof value must be set to `0x0`.

 1. type: 33703 (`reveal_alice_session_params`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `buy`] The buy `Ab`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `cancel`] The cancel `Ac`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `refund`] The refund `Ar`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `punish`] The punish `Ap`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `adaptor`] The `Ta`, an arbitrating adaptor public key
    - [`u16`: `address_len`]
    - [`address_len * byte`: `address`] The destination arbitrating address
    - [`u16`: `accordant_scalar_len`]
    - [`accordant_scalar_len * byte`: `view`] The `K_v^a`, an accordant view private key
    - [`u16`: `accordant_point_len`]
    - [`accordant_point_len * byte`: `spend`] The `K_s^a`, an accordant spend public key
    - [`u16`: `proof_len`]
    - [`proof * byte`: `proof`] The cross-group discrete logarithm zero-knowledge proof

### The `reveal_bob_session_params` Message

**Sent by**: Bob

`reveal_bob_session_params` reveals the parameters commited by the `commit_bob_session_params` message.

If not needed for the accordant blockchain the view key value must be set to `0x0`.

If not needed for the pair of Arbitrating-Accordant blockchain the proof value must be set to `0x0`.

 1. type: 33704 (`reveal_bob_session_params`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `buy`] The buy `Bb`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `cancel`] The cancel `Bc`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `refund`] The refund `Br`, an arbitrating public key
    - [`u16`: `arbitrating_point_len`]
    - [`arbitrating_point_len * byte`: `adaptor`] The `Tb`, an arbitrating adaptor public key
    - [`u16`: `address_len`]
    - [`address_len * byte`: `address`] The refund arbitrating address
    - [`u16`: `accordant_scalar_len`]
    - [`accordant_scalar_len * byte`: `view`] The `K_v^b`, an accordant view private key
    - [`u16`: `accordant_point_len`]
    - [`accordant_point_len * byte`: `spend`] The `K_s^b`, an accordant spend public key
    - [`u16`: `proof_len`]
    - [`proof * byte`: `proof`] The cross-group discrete logarithm zero-knowledge proof

### The `core_arbitrating_setup` Message

**Sent by**: Bob

`core_arbitrating_setup` sends the `lock (b)`, `cancel (d)` and `refund (e)` arbitrating transactions from Bob to Alice, as well as Bob's signature for the `cancel (d)` transaction.

 1. type: 33710 (`core_arbitrating_setup`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `lock_len`]
    - [`lock_len * byte`: `lock`] The arbitrating `lock (b)` transaction
    - [`u16`: `cancel_len`]
    - [`cancel_len * byte`: `cancel`] The arbitrating `cancel (d)` transaction
    - [`u16`: `refund_len`]
    - [`refund_len * byte`: `refund`] The arbitrating `refund (e)` transaction
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Bc` `cancel (d)` arbitrating `signature`

### The `refund_procedure_signatures` Message

**Sent by**: Alice

`refund_procedure_signatures` transmits Alice's signature for the `cancel (d)` transaction and Alice's adaptor signature for the `refund (e)` transaction. Upon receipt, Bob must validate the signatures.

 1. type: 33720 (`refund_procedure_signatures`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Ac` `cancel (d)` arbitrating `signature`
    - [`u16`: `refund_adaptor_sig_len`]
    - [`refund_adaptor_sig_len * byte`: `refund_adaptor_sig`] The `Ar(Tb)` `refund (e)` arbitrating `adaptor_signature`


### The `buy_procedure_signature` Message

**Sent by**: Bob

`buy_procedure_signature`is intended to transmit Bob's adaptor signature for the `buy (c)` transaction and the transaction itself. Upon receipt, Alice must validate the transaction and the adaptor signature.

 1. type: 33730 (`buy_procedure_signature`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `buy_len`]
    - [`buy_len * byte`: `buy`] The arbitrating `buy (c)` transaction
    - [`u16`: `buy_adaptor_sig_len`]
    - [`buy_adaptor_sig_len * byte`: `buy_adaptor_sig`] The `Bb(Ta)` `buy (c)` arbitrating `adaptor_signature`

### The `abort` Message

**Sent by**: Alice|Bob

`abort` is an `OPTIONAL` courtesy message from either swap partner to inform the counterparty that they have aborted the swap with an `OPTIONAL` message body to provide the reason.

Upon receipt, the daemon must engage the cancel path if necessary and should respond with an `abort` message.

 1. type: 33799 (`abort`)
 2. data:
    - [`swap_id`: `swap_id`] The swap unique identifier
    - [`u16`: `error_len`]
    - [`error_len * byte`: `error_body`] OPTIONAL `body`: error code | string

## References

 * [[1] Bitcoin-Monero Cross-chain Atomic Swap](https://eprint.iacr.org/2020/1126)
 * [[2] BOLT #8: Encrypted and Authenticated Transport](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md)
 * [[3] BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format)

