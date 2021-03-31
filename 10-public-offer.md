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

Public offers carry data about two specific blockchain, those data need to be interpreted in their blockchain context. A serialized binary value for a timelock can only be interpreted if the blockchain for which the value has been serialized is known. A timelock value for Bitcoin, e.g. 4 bytes unsigned interger for `nSequence`, might be interpreted differently than an Etherum timelock value. The parser must then be generic and may fail to interpret blockchain specific data if the wrong blockchain is used.

A public offer as of version 1 contains the following fields:

 * Public offer magic bytes and **version** format
 * **Network** to used on both assets to perform the swap
 * The **arbitrating asset** identifier
 * The **accordant asset** identifier
 * An arbitrating **asset amount** exchanged
 * An accordant **asset amount** exchanged
 * The definition of the **cancel timeout**
 * The definition of the **punish timeout**
 * The definition of a **fee strategy**
 * The future **maker swap role**
 * TODO add peer connection parameters

This version 1 is the simplest offer possible, it does not contain asset amount ranges to be traded nor room for price negotiation.

### Version

The public offer version contains six magic bytes and two bytes for the version and features. The bytes used for the vesion contains also feature flags, this RFC describes the version 1 and the list of features is declared as an empty list, so no flags are expected, leading the two bytes to have the value `1` after deserializing them.

### Network

Three network are defined to scope the swap:

 * Mainnet
 * Testnet
 * Local

Only the mainnet network is used to swap valuable assets, the other networks are for test purposes only and do not in any circumstances move real value.

### Asset identifiers

An asset is identified based on the [BIP44/SLIP44 [1,2]](#references), the testnet identifier is not considered valid and must be rejected by implementations.

### Amounts

Amounts must represent the value in its native smaller granularity format or are otherwise considered as invalid. For example, Bitcoin amounts must be expressed in `satoshi` and Monero amounts in `monerujo`.

### Timeouts

Timeout values are interpreted based on the arbitrating chain. For example, if Bitcoin is used as the arbitrating blockchain, the values must represent the `nSequence` input field of the transactions with a `CHECKSEQUENCEVERIFY` opcode. For other blockchains, the value might be interpreted differently.

### Fee strategy

Two fee strategy are defined:

 * Fixed
    * Contain one value, interpreted as the fixed fee to apply
 * Range
    * Contain two values: (1) minimum [inclusive], and (2) maximum [inclusive]

A fixed fee strategy will always apply the same fee on every transaction. A range strategy allows users to define a minimum and a maximum of fee to apply on every transaction. Values are interpreted based on the arbitrating blockchain. For example in Bitcoin values must define how many satoshis per virtual byte the fee should use.

### Future swap role

As defined in [01. High Level Overview](./01-high-level-overview.md) two swap roles are specified:

 * Alice
 * Bob

A swap as defined in Farcaster always involves one and only one Alice and one and only one Bob. Defining the future maker swap role is enough to derive the future taker swap role.

## Serialization

A public offer MUST follow the specified format below to be considered as valid.

 * Magic bytes is an array of bytes: `[0x46, 0x43, 0x53, 0x57, 0x41, 0x50]`, corresponding to `FCSWAP` in ASCII
 * The version and list of activated features as a two bytes unsigned little endian integer, currently set as 1, i.e. `[0x01, 0x00]`
 * The network as one byte: `0x01` for mainnet, `0x02` for testnet, and `0x03` for local
 * The arbitrating asset identifier followed by the accordant identifier, two four bytes unsigned integer serialized in little endian
 * The arbitrating asset amount followed by the accordant amount
    * one unsigned integer byte defining the number of following bytes to parse
    * an array of bytes `[bytes]` representing the amount in the smallest granular native unit for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The cancel timeout followed by the punish timeout
    * one unsigned integer byte defining the number of following bytes to parse
    * an array of bytes `[bytes]` representing the timelock value for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The fee strategy as one byte:
    * `0x01` for the fixed fee, followed by
        * one unsigned integer byte defining the number of bytes to parse per value
        * an array of bytes `[bytes]` representing the value of the fixed fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
    * `0x02` for the range fee, followed by
        * one unsigned integer byte defining the number of bytes to parse per value
        * an array of bytes `[bytes]` representing the value of minimum fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
        * one unsigned integer byte defining the number of bytes to parse per value
        * an array of bytes `[bytes]` representing the value of maximum fee interpreted for its respective blockchain; bytes are serialized with the native blockchain consensus rules.
 * The future maker swap role as one byte: `0x01` for Alice and `0x02` for Bob

```
< [0x46, 0x43, 0x53, 0x57, 0x41, 0x50] MAGIC BYTES > < [u16] version > < [u8] network >
< [u8; 4] arbitrating identifier > < [u8; 4] accordant identifier >
< [u8] len > < [u8; len] arbitrating amount value >
< [u8] len > < [u8; len] accordant amount value >
< [u8] len > < [u8; len] cancel timeout value >
< [u8] len > < [u8; len] punish timeout value >
< [u8] fee strategy > < [u8] len > < [u8; len] fixed or minimum value > (< [u8] len > < [u8; len] maximum value >)
< [u8] future maker role >
```

### Amounts

For Bitcoin, amounts must be serialized as an 8-byte unsigned little endian integer representing the number of satoshi, i.e. 1e-8 bitcoin.

For Monero, amounts must be serialized as an 8-byte unsigned little endian integer representing the number of piconero, i.e. 1e-12 monero.

### Timelocks

For Bitcoin, a timelock value must be serialized as an 4-byte unsigned little endian integer representing the `nSequence` field in the transaction and the number to push on the witness stack with a `CHECKSEQUENCEVERIFY` opcode.

## References

 * [[1] BIP 44: Multi-Account Hierarchy for Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
 * [[2] SLIP 44: Registered coin types for BIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md#slip-0044--registered-coin-types-for-bip-0044)
