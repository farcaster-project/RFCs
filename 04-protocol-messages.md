<pre>
  State: draft
  Created: 2021-01-20
</pre>

# 04. Protocol Messages

## Overview

This RFC specifies the inter-daemon communication messages, segregated by protocol swap phase roles (see [01. High Level Overview](./01-high-level-overview.md)). These messages are based on the protocol execution specified in *'Bitcoin-Monero Cross-chain Atomic Swap'* [[1](#references)] with better optimization and greater generalization over blockchains in mind.

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

The inter-daemon communication must follow *'BOLT #8: Encrypted and Authenticated Transport'* [[2](#references)] standard from the Lightning Network specifications.

All communications between nodes is encrypted in order to provide confidentiality for all transcripts between nodes and is authenticated in order to avoid malicious interference. Each node has a known long-term identifier that is a public key on Bitcoin's `secp256k1` curve. This long-term public key is used within the protocol to establish an encrypted and authenticated connection with peers.

## Messages

The inter-daemon messages must follow the *'Type-Length-Value Format'* defines in *'BOLT #1: Base Protocol'* [[3](#references)] standard from the Lightning Network specifications.

The following convenience types extend 'BOLT #1' fundamental types:

 * `swap_id`: a 32-byte swap_id

### The `commit_alice_session_params` Message

**Send by**: Alice

`commit_alice_session_params` forces Alice to commit to the result of her cryptographic setup before receiving Bob's setup. This is done to remove adaptive behavior.

 1. type: 33701 (`acommit`)
 2. data:
    - [`sha256`: `buy`] Commitment to `Ab` curve point
    - [`sha256`: `cancel`] Commitment to `Ac` curve point
    - [`sha256`: `refund`] Commitment to `Ar` curve point
    - [`sha256`: `punish`] Commitment to `Ap` curve point
    - [`sha256`: `adaptor`] Commitment to `Ta` curve point
    - [`sha256`: `view`] Commitment to `k_v^a` scalar
    - [`sha256`: `spend`] Commitment to `K_s^a` curve point

### The `commit_bob_session_params` Message

**Send by**: Bob

`commit_bob_session_params` forces Bob to commit to the result of his cryptographic setup before receiving Alice's setup. This is done to remove adaptive behavior.

 1. type: 33702 (`bcommit`)
 2. data:
    - [`sha256`: `buy`] Commitment to `Bb` curve point
    - [`sha256`: `cancel`] Commitment to `Bc` curve point
    - [`sha256`: `refund`] Commitment to `Br` curve point
    - [`sha256`: `adaptor`] Commitment to `Tb` curve point
    - [`sha256`: `view`] Commitment to `k_v^b` scalar
    - [`sha256`: `spend`] Commitment to `K_s^b` curve point

### The `reveal_alice_session_params` Message

**Send by**: Alice

`reveal_alice_session_params` reveals the parameters commited by the `commit_alice_session_params` message.

 1. type: 33703 (`areveal`)
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

### The `reveal_bob_session_params` Message

**Send by**: Bob

`reveal_bob_session_params` reveals the parameters commited by the `commit_bob_session_params` message.

 1. type: 33704 (`breveal`)
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

### The `core_arbitrating_setup` Message

**Send by**: Bob

`core_arbitrating_setup` sends the `lock (b)`, `cancel (d)` and `refund (e)` arbritrating transactions from Bob to Alice, as well as Bob's signature for the `cancel (d)` transaction.

 1. type: 33710 (`coresetup`)
 2. data:
    - [`u16`: `lock_len`]
    - [`lock_len * byte`: `lock`] The arbitrating `lock (b)` transaction
    - [`u16`: `cancel_len`]
    - [`cancel_len * byte`: `cancel`] The arbitrating `cancel (d)` transaction
    - [`u16`: `refund_len`]
    - [`refund_len * byte`: `refund`] The arbitrating `refund (e)` transaction
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Bc` `cancel (d)` signature

### The `refund_procedure_signatures` Message

**Send by**: Alice

`refund_procedure_signatures` is intended to transmit Alice's signature for the `cancel (d)` transaction and Alice's adaptor signature for the `refund (e)` transaction. Uppon reception Bob must validate the signatures.

 1. type: 33720 (`refundproc`)
 2. data:
    - [`u16`: `cancel_sig_len`]
    - [`cancel_sig_len * byte`: `cancel_sig`] The `Ac` `cancel (d)` signature
    - [`u16`: `refund_adaptor_sig_len`]
    - [`refund_adaptor_sig_len * byte`: `refund_adaptor_sig`] The `Ar(Tb)` `refund (e)` adaptor signature


### The `buy_procedure_signature` Message

**Send by**: Bob

`buy_procedure_signature`is intended to transmit Bob's adaptor signature for the `buy (c)` transaction and the transaction itself. Uppon reception Alice must validate the transaction and the adaptor signature.

 1. type: 33730 (`buyproc`)
 2. data:
    - [`u16`: `buy_len`]
    - [`buy_len * byte`: `buy`] The arbitrating `buy (c)` transaction
    - [`u16`: `buy_adaptor_sig_len`]
    - [`buy_adaptor_sig_len * byte`: `buy_adaptor_sig`] The `Bb(Ta)` `buy (c)` adaptor signature

### The `abort` Message

**Send by**: Alice|Bob

`abort` is an `OPTIONAL` courtesy message from either swap partner to inform the counterparty that they have aborted the swap with an `OPTIONAL` message body to provide the reason.

Uppon reception the daemon must engage the cancel path if necessary and should respond with an `abort` message.

 1. type: 33799 (`abort`)
 2. data:
    - [`u16`: `error_len`]
    - [`error_len * byte`: `error_body`] OPTIONAL `body`: error code | string

## References

 * [[1] Bitcoin-Monero Cross-chain Atomic Swap](https://eprint.iacr.org/2020/1126)
 * [[2] BOLT #8: Encrypted and Authenticated Transport](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md)
 * [[3] BOLT #1: Base Protocol](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format)
