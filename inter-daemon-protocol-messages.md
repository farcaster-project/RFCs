[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)

<pre>
  State: draft
  Created: 2021-01-20
</pre>

# Protocol messages

## Overview

This RFC specifies the inter-daemon communication messages, segregated by protocol swap phase roles (see [High Level Overview]()). These messages are based on the protocol execution specified in *'Bitcoin-Monero Cross-chain Atomic Swap'* [[1](#references)] with better optimization and greater generalization over blockchains in mind.

## Table of Contents

[TOC]


## Messages

### The `commit_alice_session_params` Message

**Send by**: Alice

`commit_alice_session_params` forces Alice to commit to the result of her cryptographic setup before receiving Bob's setup. This is done to remove adaptive behavior.

#### Type
TODO

#### Data
- `b`: `sha256 hash` Commitment to `b` curve point
- `c`: `sha256 hash` Commitment to `c` curve point
- `r`: `sha256 hash` Commitment to `r` curve point
- `p`: `sha256 hash` Commitment to `p` curve point
- `T`: `sha256 hash` Commitment to `T` curve point
- `k_v`: `sha256 hash` Commitment to `k_v` scalar
- `K_s`: `sha256 hash` Commitment to `K_s` curve point

#### TLV Data

- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: b]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: c]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: r]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: p]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: T]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: k_v]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: K_s]`

### The `commit_bob_session_params` Message

**Send by**: Bob

`commit_bob_session_params` forces Bob to commit to the result of his cryptographic setup before receiving Alice's setup. This is done to remove adaptive behavior.

#### Type
TODO

#### Data
- `b`: `sha256 hash` Commitment to `b` curve point
- `c`: `sha256 hash` Commitment to `c` curve point
- `r`: `sha256 hash` Commitment to `r` curve point
- `T`: `sha256 hash` Commitment to `T` curve point
- `k_v`: `sha256 hash` Commitment to `k_v` scalar
- `K_s`: `sha256 hash` Commitment to `K_s` curve point

#### TLV Data
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: b]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: c]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: r]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: T]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: k_v]`
- `[u16: sha256_hash_len]`
- `[sha256_hash_len * byte: K_s]`

### The `reveal_alice_session_params` Message

**Send by**: Alice

`reveal_alice_session_params` reveals the parameters commited by the `commit_alice_session_params` message.

#### Type
TODO

#### Data
- `b`: `secp256k1 curve point` The buy `Ab` public key
- `c`: `secp256k1 curve point` The cancel `Ac` public key
- `r`: `secp256k1 curve point` The refund `Ar` public key
- `p`: `secp256k1 curve point` The punish `Ap` public key
- `T`: `secp256k1 curve point` The `Ta` adaptor public key
- `Address`: `base58` The destination Bitcoin address
- `k_v`: `edward25519 scalar` The `K_v^a` spend private key
- `K_s`: `edward25519 curve point` The `K_s^a` view public key
- `z`: `DLEQ proof` The cross-group discrete logarithm zero-knowledge proof

#### TLV Data
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: b]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: c]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: r]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: p]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: T]`
- `[u16: Address_len]`
- `[Address_len * byte: Address]`
- `[u16: ed25519_scalar_len]`
- `[ed25519_scalar_len * byte: k_v]`
- `[u16: ed25519_point_len]`
- `[ed25519_point_len * byte: K_s]`
- `[u16: z_len]`
- `[z_len * byte: z_b]`

### The `reveal_bob_session_params` Message

**Send by**: Bob

`reveal_bob_session_params` reveals the parameters commited by the `commit_bob_session_params` message.

#### Type
TODO

#### Data
- `b`: `secp256k1 curve point` The buy `Bb` public key
- `c`: `secp256k1 curve point` The cancel `Bc` public key
- `r`: `secp256k1 curve point` The refund `Br` public key
- `T`: `secp256k1 curve point` The `Tb` adaptor public key
- `Address`: `base58` The refund Bitcoin address
- `k_v`: `edward25519 scalar` The `K_v^b` spend private key
- `K_s`: `edward25519 curve point` The `K_s^b` view public key
- `z`: `DLEQ proof` The cross-group discrete logarithm zero-knowledge proof

#### TLV Data
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: b]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: c]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: r]`
- `[u16: secp256k1_point_len]`
- `[secp256k1_point_len * byte: T]`
- `[u16: Address_len]`
- `[Address_len * byte: Address]`
- `[u16: ed25519_scalar_len]`
- `[ed25519_scalar_len * byte: k_v]`
- `[u16: ed25519_point_len]`
- `[ed25519_point_len * byte: K_s]`
- `[u16: z_len]`
- `[z_len * byte: z_b]`

### The `core_arbitrating_setup` Message

**Send by**: Bob

`core_arbitrating_setup` sends the `lock (b)`, `cancel (d)` and `refund (e)` arbritrating transactions from Bob to Alice, as well as Bob's signature for the `cancel (d)` transaction.

#### Type
TODO

#### Data
- `lock`: `BTC transaction` The arbitrating `lock (b)` transaction
- `cancel`: `BTC transaction` The arbitrating `cancel (d)` transaction
- `refund`: `BTC transaction` The arbitrating `refund (e)` transaction
- `cancel_sig`: `ECDSA signature` The `Bc` `cancel (d)` signature

#### TLV Data
- `[u16: lock_len]`
- `[lock_len * byte: lock]`
- `[u16: cancel_len]`
- `[cancel_len * byte: cancel]`
- `[u16: refund_len]`
- `[refund_len * byte: refund]`
- `[u16: cancel_sig_len]`
- `[cancel_sig_len * byte: cancel_sig]`

### The `refund_procedure_signatures` Message

**Send by**: Alice

`refund_procedure_signatures` is intended to transmit Alice's signature for the `cancel (d)` transaction and Alice's adaptor signature for the `refund (e)` transaction. Uppon reception Bob must validate the signatures.

#### Type
TODO

#### Data
- `cancel_sig`: `ECDSA signature` The `Ac` `cancel (d)` signature
- `refund_adaptor_sig`: `ECDSA signature` The `Ar(Tb)` `refund (e)` adaptor signature

#### TLV Data
- `[u16: cancel_sig_len]`
- `[cancel_sig_len * byte: cancel_sig]`
- `[u16: refund_adaptor_sig_len]`
- `[refund_adaptor_sig_len * byte: refund_adaptor_sig]`

### The `buy_procedure_signature` Message

**Send by**: Bob

`buy_procedure_signature`is intended to transmit Bob's adaptor signature for the `buy (c)` transaction and the transaction itself. Uppon reception Alice must validate the transaction and the adaptor signature.

#### Type
TODO

#### Data
- `buy`: `BTC transaction` The arbitrating `buy (c)` transaction
- `buy_adaptor_sig`: `ECDSA signature` The `Bb(Ta)` `buy (c)` adaptor signature

#### TLV Data
- `[u16: buy_len]`
- `[buy_len * byte: buy]`
- `[u16: buy_adaptor_sig_len]`
- `[buy_adaptor_sig_len * byte: buy_adaptor_sig]`

### The `abort` Message

**Send by**: Alice|Bob

`abort` is an `OPTIONAL` courtesy message from either swap partner to inform the counterparty that they have aborted the swap with an `OPTIONAL` message body to provide the reason.

Uppon reception the daemon must engage the cancel path if necessary and should respond with an `abort` message.

#### Type
TODO

#### Data
- OPTIONAL `body`: error code | string

#### TLV DATA
- `[u16: error_len]`
- `[error_len * byte: error_body]`

## References

 * [[1] Bitcoin-Monero Cross-chain Atomic Swap](https://eprint.iacr.org/2020/1126)
