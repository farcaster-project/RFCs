# Bitcoin transactions temporal safety

## Intend
In the swap protocol publishing transactions to the bitcoin mempool (=transaction pool) reveals secret keys that are required to sweep the monero wallet. Therefore it is paramont to carefully evaluate the temporal safety bounds of publishing transactions, as they may:
- be raced by valid and pre-signed protocol's transactions that are de facto double-spends or 
- reverted by blockchain reorgs


## Protocol

Here we offer a simple protocol that may be used for publishing each transaction to the Bitcoin blockchain without puting funds at risk.


The swap participant has a temporal safety parameter, $\delta_{irreversible}$, in blocks. If a transaction is mined at block $t$, at $t + \delta_{irreversible}$ participant assumes that her transaction is irreversible. 

### Funding tx

After the atomic swap protocol initialization successfully completes, Bob can publish the funding tx. 

Bob publishes the funding tx, that is later mined at block height $t_{funding}$. 

At $t_{funding} + \delta_{irreversible}$ Bob assumes his transaction is irreversible, and may move on with the protocol execution. That is, Bob can publish his buy tx. 

### Buy tx
However he should not wait too long for publishing the buy tx as the cancelation path may become available, and then a race condition (double spend) between the swap (buy tx) and the cancelation (cancel tx) paths becomes possible. 

The safety parameter to prevent race conditions (=double spends) is $\delta_{race}$, in blocks.

The cancelation path becomes valid at time $t_{funding} + \delta_{cancel}$. Thus Bob has to publish his buy transaction the latest at 

$$t_{funding} + \delta_{cancel} - \delta_{race} $$


In sum, Bob's safety window to publish the buy tx is from 
$$t_{funding} + \delta_{irreversible}$$ 
to
$$t_{funding} + \delta_{cancel} - \delta_{race} $$

where 
$$\delta_{irreversible} < \delta_{race} < \delta_{cancel}$$

A reasonble guesstimate:
$$\delta_{race} \approx 10\times \delta_{irreversible}$$


Note that $\delta_{irreversible}$ and $\delta_{race}$ must be proportional to the amount transacted. Additionally $\delta_{race}$  is proportional to mempool congestion.

The same logic applies on the cancelation path.

### Cancel tx
Cancel tx should be published as soon as it becomes valid, that is, after 

$$t_{funding} + \delta_{cancel}$$

The block height in which cancel tx is mined is $t_{cancel}$

### Refund tx
Refund tx safety window for publication 
$$t_{cancel} + \delta_{irreversible}$$ 
to
$$t_{cancel} + \delta_{punish} - \delta_{race} $$

It can be argued that the higher bound of the refund transaction window can be extended, not deliberately and safely, but in emergency cases where Bob could not be online. The refund transaction must be published before the punish transaction gets mined, as they consume the same output from the cancel tx. 

The counter argument is that if Bob publishes the refund transaction then Alice can retrieve the secret key and will get the monero, and might as well get the bitcoin if she wins the race. 

While if she only gets the bitcoin, and the monero stays locked, Bob could setup a new swap to unlock the already locked monero (no cost for Alice) and propose a nice reward to Alice for revealing her private key. That is a fairly graceful failure.

### Punish tx
Punish tx should be published as soon as it becomes valid, that is, after: 
$$t_{cancel} + \delta_{punish}$$