# XMR-BTC atomic swap core protocol

This document intends to formally specify the core of the swap protocol
execution: the protocol states and what state transitions are permited for each
given state. This is critical to prevent unintentional though exploitable
state-space explosion by implementation choices.

It currently uses a petrinet descriptor language which we refer to as net. Its
syntax is extremely simple: `transition_name input_type0 input_type1 ... ->
output_type0 output_type1 ...`. Think of it as normal function signatures on
strongly typed languages.

During runtime each instantiated input_type gives rise to a local state. At a
given time, there is a correspondance between (a) the sum type of all
constructed input_types for the entire protocol and (b) the global protocol
state.

What does this type of formal specification gives us? 
1. facilitates correct implementation of the code -- i.e., correct by
   construction approach,
1. facilitates formal verification of the code,
1. may eventually be used to create a type-checked runtime environment by
leveraging Rust's borrow-checker to impose linear type/logic constraints. Thus
correct protocol execution may be imposed at compile time rather than at runtime
for the utmost critical parts of the software.
1. facilitates combining/reusing sub-protocols into larger protocols as
   petrinets with their composable local states are ideal for that.

## Alice
Ab: secp256k1 buy key;
Ac: secp256k1 cancel key;
Ar: secp256k1 refund key;
Ap: secp256k1 punish key;
Ta -> Ta: secp256k1 adaptor key;
Ksa -> K_s^a: ed25519 spend key;
Kva -> K_v^a: ed25519 view key;
where
    Ta = K_s^a projection over secp256k1


``` net:alice0_
keygen_a () -> ab ac Ab Ac ar ap Ar Ap kva ksa AAddress
dl_prove_a ksa -> za Ksa Ta
```

### The reveal_alice_session_params Message
Ab -> b: secp256k1 curve point The buy Ab public key
Ac -> c: secp256k1 curve point The cancel Ac public key
Ar -> r: secp256k1 curve point The refund Ar public key
Ap -> p: secp256k1 curve point The punish Ap public key
Ta -> T: secp256k1 curve point The Ta adaptor public key
AAdress -> Address: base58 The destination Bitcoin address
kva -> k_v: edward25519 scalar The K_v^a spend private key
Ksa -> K_s: edward25519 curve point The K_s^a view public key
za -> z: DLEQ proof The cross-group discrete logarithm zero-knowledge proof

``` net:reveal_alice_session_params_
send0a kva Ksa Ab Ta Ac Ar Ap AAddress za  -> kva_b Ksa_b Ab_b Ta_b Ac_b Ar_b Ap_b AAddress_b  za_b
```


``` net:alice1_
aggreg_priv_view_a kva kbv_a -> kv_a
aggreg_pub_spend_a Ksa Ksb_a -> Ks_a
priv2pub_a kv_a -> Kv_a

DLVrfyOKa Ksb_a Tb_a zb_a -> DLVrfyed_a 
DLVrfyERRa Ksb_a Tb_a zb_a -> DLFailed_a

VrfyBuyOKa lock_a buy_a Ab Bb_a -> VrfyedBuy
VrfyBuyERRa lock_a buy_a Ab Bb_a -> FailedBuy

VrfyCanclOKa lock_a cancel_a Ac Bc_a  -> VrfyedCancl
VrfyCanclERRa lock_a cancel_a Ac Bc_a  -> FailedCancl

VrfyRfndOKa VrfyedCancl cancel_a refund_a Ar Br_a -> VrfyedRfnd 
VrfyRfndERRa VrfyedCancl cancel_a refund_a Ar Br_a -> FailedRfnd

VrfysigRfndOKa VrfyedCancl Bc_a cancel_a b_sig_cancel_a -> VrfyedsigRfnd 
VrfysigRfndERRa VrfyedCancl Bc_a cancel_a b_sig_cancel_a -> FailedsigRfnd

valid0_a DLVrfyed_a VrfyedBuy VrfyedCancl VrfyedRfnd VrfyedPunish VrfyedsigRfnd -> Valid0a

EncSignRfnd_a Valid0a ar Tb_a refund_a -> refund_adaptor_sig
RecKeyRfnd_a Tb_a refund_adaptor_sig -> d' ;; currently dangling, used for monero refund
SignCancl_a Valid0a ac cancel_a -> a_sig_cancel
```

### The refund_procedure_signatures Message
a_sig_cancel -> cancel_sig: ECDSA signature The Ac cancel (d) signature
refund_adaptor_sig -> refund_adaptor_sig: ECDSA signature The Ar(Tb) refund (e) adaptor signature

``` net:refund_procedure_signatures_
send1a a_sig_cancel refund_adaptor_sig -> a_sig_cancel_b refund_adaptor_sig_b
```

``` net:alice2_
EncVrfyBuyOKa Bb_a Ta buy_a -> EncVrfyedBuy_a
EncVrfyBuyERRa Bb_a Ta buy_a -> EncFailedBuy_a
WatchlockOKa lockPublished lock -> VrfylockMined
WatchlockERRa lockPublished lock -> FaillockMined

InitXlock_a Kv_a Ks_a -> Xlock
PubXlock_a VrfylockMined Xlock -> XlockPublished

DecSig_buy_a EncVrfyedBuy_a ksa buy_adaptor_sig_a -> s1
Sign_buy_a ab buy_a -> s2
aggreg0a s1 s2 -> s12
PubBuyTx buy_a s12 -> buyPublished
```

## Bob
Bf -> Bf: secp256k1 fund key;
Bb -> Bb: secp256k1 buy key;
Bbr -> Bc: secp256k1 cancel key;
Br  -> Br: secp256k1 refund key;

Tb -> Tb: secp256k1 adaptor key;

Ksb -> K_s^b: ed25519 spend key;
Kvb -> K_v^b: ed25519 view key;

where
    Tb = K_s^b projection over secp256k1

``` net:bob0_
;; bob
keygen_b () -> kbv ksb bb bc Bb Bc bf Bf br Br BAddress
dl_prove_b ksb -> zb Ksb Tb
```

### The reveal_bob_session_params Message
Bb -> b: secp256k1 curve point The buy Bb public key
Bc -> c: secp256k1 curve point The cancel Bc public key
Br -> r: secp256k1 curve point The refund Br public key
Tb -> T: secp256k1 curve point The Tb adaptor public key
BAddress -> Address: base58 The refund Bitcoin address
kbv -> k_v: edward25519 scalar The K_v^b view private key
Ksb -> K_s: edward25519 curve point The K_s^b spend public key
zb -> z: DLEQ proof The cross-group discrete logarithm zero-knowledge proof 
``` net:reveal_bob_session_params_
send0b kbv Ksb Bb Tb Bc zb Br BAddress -> kbv_a Ksb_a Bb_a Tb_a Bc_a zb_a Br_a BAddress_a
```

```net:bob1_
aggreg_priv_view_a kva_b kbv -> kv_b
aggreg_pub_spend_b Ksa_b Ksb -> Ks_b
priv2pub_b kv_b -> Kv_b

DLVrfyOKb Ksa_b Ta_b za_b -> DLVrfyed_b 
DLVrfyERRb Ksa_b Ta_b za_b -> DLFailed_b

InitTxs_b DLVrfyed_b Ab_b Bb Ac_b Bc Ar_b Br BAddress -> lock cancel refund
Sign_cancel_b bc cancel -> b_sig_cancel
```

### The core_arbitrating_setup Message
lock -> lock: BTC transaction The arbitrating lock (b) transaction
cancel -> cancel: BTC transaction The arbitrating cancel (d) transaction
refund -> refund: BTC transaction The arbitrating refund (e) transaction
b_sig_cancel -> cancel_sig: ECDSA signature The Bc cancel (d) signature

``` net:core_arbitrating_setup_
send1b lock cancel refund b_sig_cancel -> lock_a cancel_a refund_a b_sig_cancel_a
```


``` net:bob2_
EncVrfyRfndOKb Ac Tb refund refund_adaptor_sig_b -> EncVrfyedRfnd_b
EncVrfyRfndERRb Ac Tb refund refund_adaptor_sig_b -> EncFailedRfnd_b
VrfysigCanclOKb Ac cancel a_sig_cancel_b ->Vrfyed0b
VrfysigCanclERRb Ac cancel a_sig_cancel_b -> Failed0b

valid0_b EncVrfyedRfnd_b Vrfyed0b -> Valid0b

InitBuy_b Valid0b lock -> buy
EncSignBuy_b bb Ta_b buy -> buy_adaptor_sig
RecKeyBuy_b Valid0b Ta_b buy_adaptor_sig -> d
Publock_b Valid0b lock -> lockPublished

WatchXlockOKb XlockPublished Kv_b Ks_b -> VrfyXLockMined
WatchXlockERRb XlockPublished Kv_b Ks_b -> FailXLockMined

```
### The buy_procedure_signature Message
buy -> buy: BTC transaction The arbitrating buy (c) transaction
buy_adaptor_sig -> buy_adaptor_sig: ECDSA signature The Bb(Ta) buy (c) adaptor signature

``` net:buy_procedure_signature_
send2b VrfyXLockMined buy buy_adaptor_sig -> buy_a buy_adaptor_sig_a
```

``` net:bob3_
WatchbuyOKb buyPublished buy -> VrfybuyMined
WatchbuyERRb buyPublished buy -> FailbuyMined

RecSig0b VrfybuyMined buy -> s1_b _s2_b
Rec0b s1_b d -> ksa_b
aggreg_ks_b ksa_b ksb -> ks_b

```
