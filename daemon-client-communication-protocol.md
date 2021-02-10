# Daemon-client communication protocol 
[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)

<pre>
  State: draft
  Created: 2021-02-03
</pre>

This RFC specifies the messages exchanged between a user's client and their own daemon.

```
  sk,pk       instructions     pk
 -----------  ------------> -------------
 | client  |                | daemon    |
 -----------  <-----------  -------------
              state digests
```
As sketched in the above graphic, the client->daemon route consists of instructions from the client to daemon that lead to a state transition of an ongoing swap, and the daemon->client route consists of state digests sent to the client whenever a new choice becomes available to the client.

### Security considerations
> [name=Lederstrumpf][color=violet] Possibly, this has been written elsewhere already, or would find a better home in another RFC

From a security perspective, an import distinction between the client and the daemon is that the daemon only knows public keys - private keys are the privy treasure of the client. Nonetheless, the daemon MUST be viewed as a trusted component, since it exclusively verifies the correctness of the counterparty's data, controls the swap state, and can misreport progression of the swap to the client whenever the progress does not require a signature from the client. For instance, if the client is Bob who initially owns BTC in a swap, and the refund path is invoked, if the client signs the `BTX_spend` transaction and instructs the daemon to relay it, a malicious daemon could abstain from relaying it, resulting in a loss of funds for Bob, if he does not detect this error and submit the signed transaction via an alternate route before Alice can submit the `BTX_claim` transaction to punish Bob.

## Instructions

There are three types of instructions:
1. Cryptographic material
   - signatures (partial and finalized)
   - keys (public keys and Monero private view keys)
   - MuSig2 protocol messages
   - Zero knowledge proofs of DEL across groups
2. Transaction messages following [PSBT standard](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md) (`TODO`: decide on whether we actually want to follow this standard)
3. Control flow messages
   - accept a step in the swap process
   - swap cancel by the user
   
### Cryptographic material
#### 1. Initialization step
These are the messages that the client can send to the daemon in context of the initialization step.
##### `send_bitcoin_primary_pub`
Provides the daemon with the client's primary Bitcoin public key, used by Alice and Bob as the destination address for a success swap or the address used for initally locking the funds respectively.
###### Type
`void | bool (message delivery status)`
###### Data
- `B_{a, b}`: `base58`
##### `send_bitcoin_recovery_pub`
Provides the daemon with the client's Bitcoin public key, used for the recovery path.
###### Type
`void | bool (message delivery status)`
###### Data
- `B_{a, b}_r`: `base58`
##### `send_monero_private_view_key`
Provides the daemon with the client's private view key share, needed by the counterparty daemon for calculating the full private view key $K^v$.
###### Type
`void | bool (message delivery status)`
###### Data
- `k_{a, b}_v`: `edward25519` scalar
##### `send_monero_public_spend_key`
Provides the daemon with the client's public spend key, needed by the counterparty daemon for calculating the full public spend key $K^s$ and verifying the DLEQ proof. 
###### Type
`void | bool (message delivery status)`
###### Data
- `K_{a, b}_s`: `edward25519` curve point
##### `send_dleq_proof`
Provides the daemon with the client's DLEQ proof for the equal discrete logarithm of the client's Bitcoin key $B_i$ and their Monero public spend key share $K^s_i$
###### Type
`void | bool (message delivery status)`
###### Data
- `dleq_proof`: `DLEQ proof`
##### `send_secret`
Only called by Bob's daemon: provides Bob's daemon with Bob's secret. This secret is passed on to Alice's daemon only in hashed form initially so that she can verify the success path in the `SWAPLOCK` script, and then shared directly with Alice's daemon once Alice has locked her Monero so that Alice can call the success path of the script, leaking her Monero spend key share $K^s_a$ to Bob.
###### Type
`void | bool (message delivery status)`
###### Data
- `h_s`: `U256`

> [name=Lederstrumpf][color=violet] consider compressing all of the above to single initialization messages for Alice and Bob (differing by the secret only). This would be consistent with the inter-daemon specification, but diverge from h4sh3d's draft with segregated types, thus should make a decision on whether to have this granular or compressed.

#### 2. Bitcoin Locking Step
##### `send_signed_btx_refund_bob`
Only called by Bob's daemon: provides Bob's daemon with a signature on the unsigned `BTX_refund` transaction provided by the daemon via (`TODO`: link applicable state digest)

###### Type
`void | bool (message delivery status)`
###### Data
`sig_btx_refund_bob`: `ECDSA signature`
##### `send_signed_btx_spend_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_spend` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
`void | bool (message delivery status)`
###### Data
`sig_btx_spend_alice`: `ECDSA signature`
##### `send_signed_btx_refund_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_refund` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
`void | bool (message delivery status)`
###### Data
`sig_btx_refund_alice`: `ECDSA signature`
#### 3. Monero Locking Step
##### `send_signed_btx_buy_bob`
Only called by Bob's daemon: provides Bob's daemon with a signature on the unsigned `BTX_buy` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
`void | bool (message delivery status)`
###### Data
`sig_btx_buy_bob`: `ECDSA signature`
##### `send_signed_xtx_lock_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `XTX_lock` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
`void | bool (message delivery status)`
###### Data
`sig_xtx_lock_alice`: `EdDSA signature`

#### 4. Swap Step
##### `send_signed_btx_buy_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_buy` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
`void | bool (message delivery status)`
###### Data
`sig_btx_buy_alice`: `ECDSA signature`

## State digests
State digests are messages sent by the daemon to the client to relay all information needed to construct the client interface and update the swap state. These messages both allow the client to display the correct actions a user can perform and create the necessary instructions consumed daemon to continue the swap protocol.