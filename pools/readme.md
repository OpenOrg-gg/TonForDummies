*(Written and designed by [@adamscochran](https://x.com/adamscochran), please reach out with any suggestions for code improvements)*

**Disclaimer:** *These are novel design proposals, they have not been tested or audited. They aim to serve as a way for EVM/SVM devs to learn thinking about optimized TVM design structure. They should not be used in a production environment.*

The following are (non-production ready) examples of pools in FunC for the TON blockchain.

They are deisgned to theory around scaling challenges on TON.

The largest constriction point that most DAPPS face on TON is the number of wallet messages sent per minute (roughly 2). Given that TON uses async messaging and what EVM/SVM devs typically think of as "a transaction" can take place across multiple messages, dapps either must wait for a flow to finish processing, or implement smart scaling methods or queues to ensure there is no state change during the flow of a transaction.

With "Jettons" (the standard ERC-20 equivelent) the P2P scalability is solved by balances not being stored in a master contract (like an ERC-20) but instead in individual wallets associated with each user.

For example, if a user holds "TokenA" "TokenB" and "TokenC" the user has 3 distinct Jetton wallets. Each references a Jetton Master/Minter wallet that is the tokens "address" (which is how contracts can verify the authenticity of a token).

When User1 wants to send TokenA to User2, the transaction happens between their TokenA wallets without invoking the Master/Minter contract of the Jetton. Since these transactions are peer-to-peer, we do not need to worry about sequence of flow through a high-touchpoint contract.

And, because they are distinct contracts, if there are lots of users all sending transactions from their TokenA wallets, those TokenA wallets (which are their own autonomous smart contracts) could be split across many shards. Since shards only need to process data on their shard they can scale in a horizontal fashion. But, if all transactions went through a single contract like an ERC-20, then all transactions could only take place on a single shard.

This brings us to the challenge for DeFi apps. 

Historically the precedent on EVM/SVM systems is to have "pools" or "LP tokens" or "markets" that are single contracts that track balances, rates, state, etc in a unified way. For example, having a token in a Uniswap LP pool and performing a swap.

On TON with the messaging flow taking up to 30 seconds and being multi-step async, a monolithic pool would have basically know choice but to only settle 2 tx/minute.

This is what takes place in [V1 our "simple pool"](https://github.com/OpenOrg-gg/TonForDummies/blob/master/pools/v1-simple.func) which is basically a rewriting of the standard EVM AMM into FunC.

In V2, we take a different approch [V2 "batched pool"](https://github.com/OpenOrg-gg/TonForDummies/blob/master/pools/v2-batched-tick.func) this pool instead queues all incoming transactions and adds them to batches. A "batch" can hold up to 4 messages at once, so this is up to 8x/minute (still not efficient). Rather than count the messages, we also use a tick-tock function to execute the batches every 5 minutes which isn't recommended for production. Currently, only specially registered contracts can be invoked by the native Tick/Tock features on TON's TVM, but implementing a feature in this model can be triggered by an automated keeper. In the future it may be possible to set up a gas pool and register your contract for automated tick/tock execution, which is why we've confirmed to this convention.

There are a few more implementations we can move to next:

* High-load wallets
* Segmenting
* Dynamic Segmenting/Balancing
* Multi-pool structure
* Multi-router structure
* Optimistic execution w/ Delayed Settlement (not recommended)
* Commit ordering schemes (not recommended)
* Quoted vs actual settlement (really not recommended)

In our [V3 Segemented Shared Rebalancing with Highload Wallet Wrapper](https://github.com/OpenOrg-gg/TonForDummies/blob/master/pools/v3-Segemented-Shared-Rebalancing-AMM-with-Highload-Wallet-Wrapper.func) we implement a few of these approaches.


### Segmentation:

First is *segmenting*

In TON since a transactions flow takes place across multiple async messages, a pool can find itself stuck waiting on a messaging flow to finish (any time a user swaps, adds or removes) as it needs to be able to confidently know the final state of the transaction and update balances before starting on the next transaction.

However, internally we can decide to map our liquidity across multiple segments. These are essentially X number of mini internal pools that we store balances across and store the state of distinctly.

We can either keep sequential count of which segment we're using for the next transaction, or, even try and implement more advanced solvers to spread liquidity across multiple segments when required.

In our sample, we store the segments as a struct with an array of structs to track all segments:

```
;; Segment structure
struct Segment {
    int balance_a;
    int balance_b;
    int total_supply;
}

;; Array of segments
global Segment[] segments;
```

and we maintain a global state of the collective values of all the segments:

```
;; Storage variables
global int balance_a;  ;; Total balance of token A
global int balance_b;  ;; Total balance of token B
global int total_supply;  ;; Total supply of liquidity tokens
global int last_rebalance_time;  ;; Timestamp of the last rebalance
global int last_segment_manage_time;  ;; Timestamp of the last segment management
global int active_segments;  ;; Number of active segments
```

In our swaps, we specifically iterate through the segments sequentially, making a transaction with only one segment at a time:

```
;; Swap tokens within a specific segment
int swap(int segment_index, int amount_in, int token_in_index) impure {
    require(segment_index < active_segments, "Invalid segment index");
    Segment segment = segments[segment_index];
    
    int balance_in = token_in_index == 0 ? segment.balance_a : segment.balance_b;
    int balance_out = token_in_index == 0 ? segment.balance_b : segment.balance_a;
    
    int amount_out = (amount_in * balance_out) / (balance_in + amount_in);
    
    ;; Transfer tokens using high-load wallet wrappers
    if (token_in_index == 0) {
        receive_jettons(token_a_wrapper_address, amount_in);
        interact_with_wrapper(token_b_wrapper_address, OP_BURN_AND_RELEASE, amount_out, msg.sender);
        segment.balance_a += amount_in;
        segment.balance_b -= amount_out;
    } else {
        receive_jettons(token_b_wrapper_address, amount_in);
        interact_with_wrapper(token_a_wrapper_address, OP_BURN_AND_RELEASE, amount_out, msg.sender);
        segment.balance_b += amount_in;
        segment.balance_a -= amount_out;
    }
    
    segments[segment_index] = segment;
    return amount_out;
}
```

Our tick-tock set up, called by a Keeper, then checks the numbers of active segments and adjusts if required (although more advanced checks and math should be used)

```
;; Manage segments (called every hour)
() manage_segments() impure {
    if (now() - last_segment_manage_time >= SEGMENT_MANAGE_INTERVAL) {
        int avg_liquidity = total_supply / active_segments;
        int threshold = avg_liquidity / 2;
        
        if (avg_liquidity > threshold * 2 & active_segments < MAX_SEGMENTS) {
            active_segments += 1;
        } elseif (avg_liquidity < threshold & active_segments > MIN_SEGMENTS) {
            active_segments -= 1;
        }
        
        last_segment_manage_time = now();
    }
}
```

After which we call the `rebalance_segments()` function to balance liqudity across the total amount of segments. (Once again, calculations should be added for production to optimize liquidity across segments, ideally performed by an offchain keeper and input into your tick-tock function, but can be done onchain)

In our base instance where we start of declaring 10 segments, this now means we can do 10x batches, so we're up to 80x transactions a minute.

The dynamic scaling of our segments, means as long as we have sufficient liquidity, we can continue to scale up the number of transactions per minute we can do.

### High-load Wallets:

The next upgrade we can use is high-load wallets. A high-load wallet is a batch processing wallet, that can contain 254 transactions per mesage.

But these wallets, only work if the apps and wallets they are interacting with are compatible.

Since the standard Jetton does not make use of a high-load wallet, we're limited in what we can achieve when interacting with these tokens.

To avoid this, I've proposed an [Internal High-Load Jetton Wrapper](https://github.com/OpenOrg-gg/TonForDummies/blob/master/pools/V3-Example-Jetton-Highload-Wrapper-Internal.func) which allows an AMM to wrap a Jetton as a high-load wallet that can only be managed by that specific AMM.

This will allow at least for the depositing, swapping and settlement of trades to be batched via high-load, so that in a worst case scenario a user is waiting on at most a queued transaction for settling the wrapped Jetton into the native Jetton. (Although there are some efforts to push for the Jetton standard to natively be high-load, it's unlikely to be successful)

```
;; Action structure
struct Action {
    int op;        ;; Operation type (transfer, burn, internal_transfer)
    int amount;    ;; Amount of tokens
    slice to;      ;; Destination address
    slice from;    ;; Source address (for internal transfers)
}
```

The internal action queue holds the set of actions as structs ready for the next message, these are only sent from the AMM pool contract.

They could be triggered either when the max array size of 254 is reached, or on a timer basis. Either way, implementing logic for a queue that execeeds this amount or partial queue failure is important, as TON does not natively rollback state.

In this model, while you cannot likely gaurentee an execution price, you can ensure that tx are FIFO which means there is no MEV/frontrunning risk in the execution, only slippage. A user could in theory be allowed to set a max slippage as well,  which would allow you to fail that specific tx if the slippage is too great by the time their tx executes. This has UX tradeoffs that each DAPP must consider.

### Overall Performance:

With these different versions we can see how rapidly we can scale transactions with good design principles:

| Version  | TX Per Minute |
| ------------- | ------------- |
| V1  | 2  |
| V2  | 8 |
| V3 w/ 10 Segments  | 80 |
| V3 w/ 100 Segments & HL Wallets  | 204,800 |

It's likely that even in V3 with many segments and HL wallets you'll hit other design snags, such as needing to create a sharded design approach for the router rather than having a monolithic router.

But by using segments and HL wallets, your scale isn't restricted by the throughput of the chain, but instead by the total amount of liquidity you can maintain relative to the user swaping experience. (i.e. too low a liquidity across many segments means bad slippage)

Since liquidity should scale in a fairly linear fashion with your overall user demand, this scaling model should allow an AMM to maintain performance.

Where there still may be challenges to solve is the rare cases of high-tx AMM pools with low liquidity that still want low slippage. This is likely most prevelant in early launch memecoin pools that we've seen on other chains, and presents its own set of design challenges.
