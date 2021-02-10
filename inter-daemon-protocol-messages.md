# Inter-daemon protocol messages
[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)


<pre>
  State: draft
  Created: 2021-01-20
</pre>

[TOC]

This RFC specifies the inter-daemon communication messages, segregated by Bob's and Alice's daemons. These messages follow the protocol execution specified in [Bitcoin--Monero Cross-chain Atomic Swap](https://github.com/h4sh3d/xmr-btc-atomic-swap/blob/master/whitepaper/xmr-btc.pdf). The applicable swap phases are documented in [User Stories / High Level Protocol](/pym9JPVlRK-RfQGOUv26aQ).

## Bob
### `send_initialization_parameters_bob`
`send_initialization_parameters` provides Alice with the Bobs's Bitcoin addresses ($B_b$, $B_b^s$, $B_b^r$), Monero keys ($k_b^v$, $K_b^s$), $z_b$, and $h_s$

#### Type
`void | bool (message delivery status)`
> [name=Lederstrumpf][color=violet] In my view, this should be void, unless we return the success/failure of message delivery
#### Data
- `B_b`: `base58`
- `B_b_s`: `base58`
- `B_b_r`: `base58`
- `k_b_v`: `edward25519 scalar`
- `K_b_s`: `edward25519 curve point`
- `z_b`: `DLEQ proof`
- `h_s`: `SHA256`

### `send_bitcoin_transactions`
`send_unsigned_bitcoin_transactions` sends the lock, refund and spend transactions from Bob to Alice, as well as Bob's signature for the refund transaction.
#### Type
`void | bool (message delivery status)`
#### Data
- `BTX_lock`: `BTC transaction`
- `BTX_refund`:`BTC transaction`
- `BTX_spend`:`BTC transaction`
- `sig_btx_refund_bob`: `ECDSA signature`
### `send_bitcoin_buy_transaction`
`send_bitcoin_buy_transaction` sends the [`BTX_buy`](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA#Buy) transaction with Bob's signature for it
#### Type
`void | bool (message delivery status)`
#### Data
- `BTX_buy`:`BTC transaction`
- `sig_btx_buy_bob`: `ECDSA signature`
### `send_bitcoin_buy_secret`
`send_bitcoin_buy_secret` sends the secret `s` Alice requires to trigger the `TRUE` branch of the `SWAPLOCK` script
#### Type
`void | bool (message delivery status)`
#### Data
- `s`: `U256`
## Alice
### `send_initialization_parameters_alice`
`send_initialization_parameters` provides Bob with Alice's Bitcoin addresses ($B_a$, $B_a^s$, $B_a^r$), Monero keys ($k_a^v$, $K_a^s$), and $z_a$ 
#### Type
`void | bool (message delivery status)`
#### Data
- `B_a`: `base58`
- `B_a_s`: `base58`
- `B_a_r`: `base58`
- `k_a_v`: `edward25519 scalar`
- `K_a_s`: `edward25519 curve point`
- `z_a`: `DLEQ proof`
### `send_bitcoin_transaction_signatures`
`send_bitcoin_transaction_signatures` provides Bob with Alice's signatures of [`BTX_refund`](https://hackmd.io/YfMko2WPR9iITsw4MsLcPA#Refund) and `BTX_spend`
#### Type
`void | bool (message delivery status)`
#### Data
- `signature_refund_alice`: `ECDSA signature`
- `signature_spend_alice`: `ECDSA signature`
## Either party
### Abort
`abort` is an `OPTIONAL` courtesy message from either swap partner to inform the counterparty that they have aborted the swap with an `OPTIONAL` message body to provide the reason
#### Type
`error`
#### Data
- OPTIONAL `body`: error code | string
