# OmniBOLT #5: Atomic Swap Protocol among Channels

>*An atomic swap is a smart contract technology that enables the exchange of one cryptocurrency for another without using centralized intermediaries, such as exchanges.*

> -- https://www.investopedia.com/terms/a/atomic-swaps.asp
  
In general, atomic swaps take place between different block chains, for exchanging tokens with no trust of each other. Channels defined in OmniBOLT can be funded by any token issued on OmniLayer. If one needs to trade his tokens, say USDT, for some one else's Bitcoins, both parties are required to acknowledge receipt of funds of USDT and BTC, within a specified timeframe using a cryptographic hash function. If one of the involved parties fails to confirm the transaction within the timeframe, then the entire transaction is voided, and funds are not exchanged, but refunded to original account. The latter action ensures to remove counterparty risk.  

The standard swap procedure between channels is:

```
     Alice in channel                                             Bob in channel
[Alice ---900 USDT---> Bob]                                 [Alice <---1 BTC--- Bob]
    +----------------+                                           +----------------+
    |                |                                           |                |
    |     create     |                                           |                |
    |      Tx 1      |----(1)---  tell Bob Tx 1 created   -----> |     create     |
    |                |                                           |      Tx 2      | 
    |                |                                           |                |
    |     Locked     |<---(2)--  Acknowledge and create Tx 2 --- |     Locked     |
    |       by       |                                           |       by       |
    |     Hash(R),   |                                           |      Hash(R)   |
    | Time-Locker t1 |                                           | Time-Locker t2 |
    |       and      |                                           |       and      |
    |   Bob's Sig    |                                           |   Alice's Sig  |
    |                |                                           |     t2 < t1    |
    |                |                                           |                |
    |                |                                           |                |
    |                |                                           |                |
    |                |----(3)----   Send R to get BTC     -----> | Alice + 1 BTC  |
    | Bob + 900 USDT |<---(4)----   Send R to get USDT    ------ |                |
    |                |                                           |                |
    |                |                                           |                |
    |                |----(5)----     or time out,               |                |
    |                |		   refund to both sides   -----> |                |
    |                |                                           |                |
    +----------------+                                           +----------------+

    - where Tx 1 transfers 1000 USDT to Bob in channel `[Alice, USDT, Bob]`, locked by Hash(R), t1 and Bob's signature. 
    - Tx 2 transfers 1 BTC to Alice in channel `[Alice, BTC, Bob]`, locked by Hash(R), t2(`t2 < t1`) and Alice's signature . 
    

```  
  

## Hashed TimeLock Swap Contract

Hashed TimeLock Swap Contract (HTLSC) consists of two seperate HTLCs with extra specified exchange rate of tokens and time lockers.  

Simply there are 5 steps in a swap. In step 3, Alice sends R to Bob, hence she can unlock transaction 2 to get her 1 BTC in the channel `[Alice, BTC, Bob]`. Therefor Bob knows R, and use R to unlock his 900 USDT in the channel `[Alice, USDT, Bob]`.  

No participant is able to cheat. After inputting R in each channel, the transaction 1 and 2 turn into general commitment transactions, which is the same procedure that how an [HTLC transforms to a commitment transaction](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#terminate-htlc-off-chain).

In channel `[Alice, USDT, Bob]`, Alice create an HTLC and its mirror transactions on Bob side, with time locker `t1`, which in the diagram is 3 days as an example.

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png">
</p>

At the same time, Bob creates another HTLC in the channle `[Alice, BTC, Bob]` and its mirror transactions on Alice side, sending the agreed number of BTCs to Alice. Time locker `t2` is set to be 2 days, less than `t1=3` days.

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy-BTC-channel.png">
</p>



## `swap`

`swap` specifies the asset that one peer needs to transfer.

1. type: -80 (swap)
2. data:
  * [`channel_id`:`channel_id_from`]: Alice initiate the swap procedure by creating an HTLSC.
  * [`channel_id`:`channel_id_to`]: Bob who Alice want to trade token with.
  * [`u64`:`property_sent`]: Omni asset (id), which is sent out to Bob.
  * [`u64`:`property_receieved`]: the Omni asset (id), which is required to the counter party (Bob) 
  * [`u64`:`amount`]: ammout of the property that is sent.
  * [`u64`:`exchange_rate`]: `= property_sent/property_receieved`. For example, sending out 900 USDT in exchange of 1 BTC, the exchange rate is 900/1.
  * [`u64`:`transaction_id`]: HTLSC transaction ID, which is the one sending asset in `channel_id_from`. 
  * [`u64`:`hashed_R`]: Hash(R).     
  * [`u64`:`time_locker`]: For example, 3 days. 
 

## `swap_accepted`

`swap_accepted` specifies the asset that one peer needs to transfer.

1. type: -81 (swap_accepted)
2. data:
  * [`channel_id`:`channel_id_from`]: Alice initiate the swap procedure by creating an HTLSC.
  * [`channel_id`:`channel_id_to`]: Bob who Alice want to trade token with.
  * [`u64`:`property_sent`]: Omni asset (id), which is sent out to Bob.
  * [`u64`:`property_receieved`]: the Omni asset (id), which is required to the counter party (Bob) 
  * [`u64`:`amount`]: ammout of the `property_receieved` that is sent in `channel_id_to`.
  * [`u64`:`exchange_rate`]: `= property_sent/property_receieved`. For example, sending out 900 USDT in exchange of 1 BTC, the exchange rate is 900/1.
  * [`u64`:`transaction_id`]: HTLSC transaction ID, which is the one sending asset in `channel_id_to`. 
  * [`u64`:`hashed_R`]: Hash(R).     
  * [`u64`:`time_locker`]: For example, 2 days, which must be less than the `time_locker` in message `swap`. 

Bob in `channel_id_to` has to monitor the `transaction_id` in channel `channel_id_from`, to see whether or not the corresponding transactions, like RD, HED, HTRD, etc, have been correctly created. After he validates the transactions, he will create HTLSC according to the arguments `amount` in channel `channel_id_to`, and then reply Alice with message `swap_accepted`.


Alice receives the message `swap_accepted`. If anything is not exactly correct, Alice will not send R to get her assets in `channel_id_to`, hence Bob is not able to get this asset in `channel_id_from`. After a timeframe, the two channels revock to their previous state.

## Remark
Atomic swap is a foundation of lots of blockchain application. Next chapter will see some examples, which are intuitive and will help our readers to build much more complex applications for real world businesses. 


 
 