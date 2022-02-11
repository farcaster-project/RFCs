<pre>
  State: draft
  Created: 2021-3-23
</pre>

# 10. Public Offer

## Overview

This RFC describes and formalizes the content of a public offer and its serialization format. The public offer is used during the first phase for discovery and connection purposes among participants. A public offer encodes trade data and participants' data.

## Table of Contents

  * [Content](#content)
    * [Version](#version)
    * [Network](#Network)
    * [Asset identifiers](#asset-identifiers)
    * [Amounts](#amounts)
    * [Fee strategy](#fee-strategy)
    * [Future swap role](#future-swap-role)
  * [Serialization](#serialization)
    * [Amounts](#amounts)
    * [Timelocks](#timelocks)
  * [References](#references)

## Content

Public offers carry data about two specific blockchains, and these data need to be interpreted in their blockchain context. A serialized binary value for a timelock can only be interpreted if the blockchain for which the value has been serialized is known. A timelock value for Bitcoin, e.g. a type and 4 bytes unsigned integer for the `nSequence` field, might be interpreted differently than an Etherum timelock value. The parser must then be generic and may fail to interpret blockchain-specific data if the wrong blockchain is used.

A public offer as of version 1 contains the following fields:

 * Public offer magic bytes and **version** format
 * **Network** to used on both assets to perform the swap
 * **Time validity** of the offer
 * The **arbitrating asset** identifier
 * The **accordant asset** identifier
 * An arbitrating **asset amount** exchanged
 * An accordant **asset amount** exchanged
 * The definition of the **cancel timeout**
 * The definition of the **punish timeout**
 * The definition of a **fee strategy**
 * The future **maker swap role**
 * The **node address** where to connect
 * A **signature** of the offer

This version 1 is the simplest offer possible, it does not contain asset amount ranges to be traded nor room for price negotiation.

### Version

The public offer version contains six magic bytes and two bytes for the version and features. The bytes used for the version also contain feature flags. This RFC describes version 1. The list of features is declared as an empty list, so no flags are expected, leading the two bytes to have the value `1` after deserializing them.

### Network

Three networks are defined to scope the swap:

 * Mainnet
 * Testnet
 * Local

Only the mainnet network is used to swap valuable assets, the other networks are for test purposes only and do not under any circumstances move real value.

### Time validity

Time constraining the validity of an offer allows better UI/UX: offers can be discarded/hidden when their validity is expired. Public offers are designed as the entry point of a Farcaster swap but do not make any assumptions on how they are shared nor any guarantee about the maker's liveness: a non-expired offer may already have been completed by another taker.

### Asset identifiers

An asset is identified based on the [BIP44/SLIP44 [1,2]](#references), the testnet identifier is not considered valid and must be rejected by implementations.

### Amounts

Amounts must represent the value in its native smallest granularity format or are otherwise considered invalid. For example, Bitcoin amounts must be expressed in `satoshi` and Monero amounts in `piconero`.

### Timeouts

Timeout values are interpreted based on the arbitrating chain. For example, if Bitcoin is used as the arbitrating blockchain, the values may represent the `nSequence` input field of the transactions with a `CHECKSEQUENCEVERIFY` opcode. For other blockchains, the value might be interpreted differently, but it may not matter whether the transactions use relative timelocks or absolute timelocks. The value stored inside the `[bytes]` array must contain a type along the value if necessary, e.g. see Bitcoin example.

### Fee strategy

Two fee strategies are defined:

 * Fixed
    * Contain one value, interpreted as the fixed fee to apply
 * Range
    * Contain two values: (1) minimum [inclusive], and (2) maximum [inclusive]

A fixed fee strategy will always apply the same fee on every transaction. A range strategy allows users to define a minimum and a maximum fee to apply to every transaction. Values are interpreted based on the arbitrating blockchain. For example in Bitcoin values must define how many satoshis per virtual byte the fee should use.

### Future swap role

As defined in [01. High-Level Overview](./01-high-level-overview.md) two swap roles are specified:

 * the accordant seller
 * the arbitrating seller

A swap as defined in Farcaster always involves one and only one the accordant seller and one and only one the arbitrating seller. Defining the future maker swap role is enough to derive the future taker swap role.

### Node address

The node address contains a `node_id` and `remote_addr`. The format is defined in [`internet2`](#references) as a `RemoteNodeAddr`. The `node_id` is a secp256k1 public key used as an encryption/id key for the session. This public key is used to validate the signature provided at the end of the public offer.

### Signature

A signature is appended at the end of the public offer. The signature must be valid for the public key present in the node address, i.e. the `node_id`.

## Serialization

A public offer MUST follow the specified format below to be considered valid.

 * Magic bytes is an array of bytes: `[0x46, 0x43, 0x53, 0x57, 0x41, 0x50]`, corresponding to `FCSWAP` in ASCII
 * The version and list of activated features as two bytes unsigned little endian integer, currently set as 1, i.e. `[0x01, 0x00]`
 * The network as one byte: `0x01` for mainnet, `0x02` for testnet, and `0x03` for local
 * The offer validity as a "UNIX timestamp". The offer remains valid up to the timestamp value - an 8-byte signed integer serialized in little endian - and should be considered invalid after reaching this value
 * The arbitrating asset identifier followed by the accordant identifier, two four bytes unsigned integer serialized in little endian
 * The arbitrating asset amount followed by the accordant amount
    * A length prefix for the number of following bytes to parse
    * An array of bytes `[bytes]` representing the amount in the smallest granular native unit for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The cancel timeout followed by the punish timeout
    * A length prefix for the number of following bytes to parse
    * An array of bytes `[bytes]` representing the timelock value for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The fee strategy as one byte:
    * `0x01` for the fixed fee, followed by
        * A length prefix for the number of bytes to parse the value
        * An array of bytes `[bytes]` representing the value of the fixed fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
    * `0x02` for the range fee, followed by
        * A length prefix for the number of bytes to parse the value
        * An array of bytes `[bytes]` representing the value of minimum fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
        * A length prefix for the number of bytes to parse the value
        * An array of bytes `[bytes]` representing the value of maximum fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The future maker swap role as one byte: `0x01` for the accordant seller and `0x02` for the arbitrating seller
 * The node address where takers can try to connect
    * A lengh prefix for the number of bytes to parse
    * An array of bytes `[bytes]` representing the node address as an [`internet2`](#references) strict encoded `RemoteNodeAddr`
        * Node address must be provided in the format `<node_id>@<node_inet_addr>[:<port>]`, where `<node_inet_addr>` may be IPv4, IPv6, Onion v2 or v3 address
 * A signature of the previous serialized bytes, valid for the `node_id` provided in the node address

Data to sign:

```
< [0x46, 0x43, 0x53, 0x57, 0x41, 0x50] MAGIC BYTES > < [u16] version > < [u8] network >
< [i64] max age validity timestamp >
< [u8; 4] arbitrating identifier > < [u8; 4] accordant identifier >
< [u16] len > < [u8; len] arbitrating amount value >
< [u16] len > < [u8; len] accordant amount value >
< [u16] len > < [u8; len] cancel timeout value >
< [u16] len > < [u8; len] punish timeout value >
< [u8] fee strategy > < [u16] len > < [u8; len] fixed or minimum value > (< [u16] len > < [u8; len] maximum value >)
< [u8] future maker role >
< [u16] len > < [u8; len] strict encoded node address >
```

Length prefixes are 16-bits unsigned little-endian encoded integers, this allows to store up to 65535 bytes per value, which is considered enough for all the potential use cases that should be covered by this public offer serialization format. In some cases 8-bits integers would be fine as many values does not length more than 255 bytes, but using 16-bits prefixes matches the other messages formats and space efficiency is not a priority here.

Signature is performed with ECDSA over secp256k1, which is used to create the `node_id`.

### Amounts

For Bitcoin, amounts must be serialized as an 8-byte unsigned little endian integer representing the number of satoshis, i.e. 1e-8 bitcoin.

For Monero, amounts must be serialized as an 8-byte unsigned little endian integer representing the number of piconero, i.e. 1e-12 monero.

### Timelocks

For Bitcoin, a timelock value must be serialized as a one byte type and a 4-byte unsigned little endian integer value.

**Serialization**:

 * `< [u8] type >`: the type of timelock; only value `0x00` is valid and represents an `nSequence` field in the transaction and the number to push on the witness stack with a `CHECKSEQUENCEVERIFY` opcode.
 * `< [u8; 4] type >`: the numeric value to use according to the type.

## References

 * [[1] BIP 44: Multi-Account Hierarchy for Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
 * [[2] SLIP 44: Registered coin types for BIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md#slip-0044--registered-coin-types-for-bip-0044)
 * [[3] `internet2` Rust crate](https://github.com/internet2-org/rust-internet2)
