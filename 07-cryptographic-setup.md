<pre>
  State: draft
  Created: 2021-2-17
</pre>

# 07. Cryptographic Setup

## Overview

This RFC specifies the cryptographic primitives used to transfer secrets through transactions with adaptor signatures and specifies the cryptographic setup required at the beginning of a swap to guarentee funds safety.

## Table of Contents

  * [Cryptographic Keys](#cryptographic-keys)
    * [Alice](#alice)
    * [Bob](#bob)
  * [Adaptor Signatures](#adaptor-signatures)
    * [ECDSA Scripts](#ecdsa-scripts)
    * [Taproot Schnorr Scripts](#taproot-schnorr-scripts)
  * [Cross-group Discrete Logarithm Proof](#cross-group-discrete-logarithm-proof)
  * [References](#references)

## Cryptographic Keys

We use the two swap roles as described in User Stories, Alice and Bob, to describe the cryptographic keys needed by each of them.

### Alice

```
Ab: secp256k1 buy key;
Ac: secp256k1 cancel key;
Ar: secp256k1 refund key;
Ap: secp256k1 punish key;

Ta: secp256k1 adaptor key;

K_s^a: ed25519 spend key;
K_v^a: ed25519 view key;

where
    Ta = K_s^a projection over secp256k1
```

### Bob

```
Bf: secp256k1 fund key;
Bb: secp256k1 buy key;
Bc: secp256k1 cancel key;
Br: secp256k1 refund key;

Tb: secp256k1 adaptor key;

K_s^b: ed25519 spend key;
K_v^b: ed25519 view key;

where
    Tb = K_s^b projection over secp256k1
```

## Adaptor Signatures

We describe an adaptor signature interface and two instantiations, one for ECDSA inside Bitcoin scripts and one for Schnorr inside Taproot scripts.

An adaptor signature scheme extend a standard signature (`Gen`, `Sign`, `Vrfy`) scheme with:

 * `EncGen`: An encryption key generation algorithm, in this protocol the encryption key generation is linked to the cross-group DLEQ proof.

 * `EncSig`: Encrypt a signature and return an adaptor signature. This RFC uses the public key tweaking method.

 * `EncVrfy`: Verify an adaptor signature based on public parameters, if validation passes the decripted adaptor signature is a valid signature.

 * `DecSig`: Decrypt an adaptor signature by injecting the encryption key.

 * `RecKey`: Recover the key material needed for extracting the encryption key with `Rec`.

 * `Rec`: Recover the encryption key based on the adaptor signature and the decrypted signature.

### ECDSA Scripts

 * `EncSig`:

```
 pi = PDLEQ( (G, R'), (T, R), k )
 s' = k^-1 ⋅ ( hash(m) + f(R)d )
 
 return (R, R', s', pi)
 
where
    f(): x-coordinate mod q
    P = dG
    R = kT
    R' = kG
```

`PDLEQ` produces a zero-knowledge proof of knowledge of a same relation `k` between two pairs of elements in the same group, i.e. `(G, R')` and `(T, R)`.

 * `EncVrfy`:

```
 VDLEQ( (G, R'), (T, R), pi ) =? 1

 R' =? ( H(m)G ⋅ f(R)P )^{s'^-1}

where
    f(): x-coordinate mod q
    P = dG
    R = kT
    R' = kG
```

`VDLEQ` verifies a zero-knowledge proof of knowledge of a same unknown relation `x` between two pairs of elements in the same group.

 * `DecSig`:

```
 s = s't^-1
   = k^-1 ⋅ ( hash(m) + f(R)d ) ⋅ t^-1

 return (f(R), s)

where
    f(): x-coordinate mod q
    P = dG
    R = kT
    R' = kG
```

 * `Rec`:

```
 t' = s^-1 ⋅ s'
 
      /  s' if s'G = T
 t = |  -s' if s'G = T^-1
      \  NaN otherwise
```

### Taproot Schnorr Scripts

This notation follows BIP 340, with some simplication on even y coordinate check for private keys:

 * `EncSig`:

```
 s' = k + tagged_hash( bytes(R + T) || bytes(P) || m )d
```

 * `EncVrfy`:

```
 s'G =? R + tagged_hash( bytes(R + T) || bytes(P) || m )P
 
where
    P = dG
```

 * `DecSig`:

```
 s = s' + t
   = k + t + tagged_hash( bytes(R + T) || bytes(P) || m )d

where
    P = dG
    R = kG
    T = tG
```

 * `Rec`:

```
 t = s - s'
```

BIP 340 Schnorr:

 * `Vrfy`:

```
 sG =? R + tagged_hash( bytes(R) || bytes(P) || m )P
```

## Cross-group Discrete Logarithm Proof

TODO

## References

 * [[1] One-Time Verifiably Encrypted Signatures A.K.A. Adaptor Signatures](https://github.com/LLFourn/one-time-VES/blob/master/main.pdf)
 * [[2] BIP 0340: Schnorr Signatures for secp256k1](https://en.bitcoin.it/wiki/BIP_0340)
