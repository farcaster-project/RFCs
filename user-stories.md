<pre>
  State: draft
  Created: 2020-11-11
</pre>

# User Stories

We describe a basic user experience with a atomic swap GUI client for Alice and Bob.

![](https://github.com/farcaster-project/RFCs/raw/master/images/gui-mocks.jpg)

## Alice

Alice, who owns Monero, is prompted for the swap settings: amount exchanged, counterparty address, refund address, etc.

When the connection with Bob is done and the parameters are validated by both participants and the bitcoin are locked, she's asked to send her funds to the swap address. For this action she has a limited amount of time, after that the GUI will indicate her that it is no longer safe to lock the funds.

After sending the funds during the safe timelaps and after their confirmations time, she will receive the bitcoin in her address.

## Bob

Bob, who owns Bitcoin, is prompted for the swap settings, equivalent to Alice.

His send his bitcoin to an intermediary address, filling the swap walltet, after enough confirmation on the bitcoin transaction and after seeing Alice funds locked he'll see the swap as completed and has the choice of exporting the Monero wallet seeds or send the Monero to another address.
