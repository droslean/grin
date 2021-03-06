Dandelion in Grin: Privacy-Preserving Transaction Propagation
==================
This document describes the implementation of Dandelion in Grin.
## Introduction

Dandelion is a new transaction broadcasting mechanism that reduces the risk of eavesdroppers linking transactions to the source IP. Moreover, it allows Grin transactions to be aggregated (removing input-output pairs) before being broadasted to the entire network giving an additional privacy perk.

Dandelion was introduced in [1] by G. Fanti et al. and presented at ACM Sigmetrics 2017. On June 2017 a BIP was proposed introducing a more practical and robust variant of Dandelion called Dandelion++ [2].  This document is an adapation of this BIP for Grin.

## Mechanism

Dandelion transaction propagation proceeds in two phases: first the “stem” phase, and then “fluff” phase. During the stem phase, each node relays the transaction to a *single* peer. After a random number of hops along the stem, the transaction enters the fluff phase, which behaves just like ordinary flooding/diffusion. Even when an attacker can identify the location of the fluff phase, it is much more difficult to identify the source of the stem.

Illustration:
<pre>
                                                   ┌-> F ...
                                           ┌-> D --┤
                                           |       └-> G ...
  A --[stem]--> B --[stem]--> C --[fluff]--┤
                                           |       ┌-> H ...
                                           └-> E --┤
                                                   └-> I ...
</pre>

This mechanism also allows Grin transactions to be aggregated during the stem phase and then broadcasted to all the nodes on the network. This result in transaction aggregation and possibly cut-through (thus removing spent outputs) giving a significant privacy gain similar to a non-interactive coinjoin with cut-through.

## Specification

The Dandelion protocol is based on three mechanisms:

1. *Stem/fluff propagation.* Dandelion transactions begin in “stem mode,” during which each node relays the transaction to a single randomly-chosen peer. With some fixed probability, the transaction transitions to “fluff” mode, after which it is relayed according to ordinary flooding/diffusion.

2. *Stem Mempool.* During the stem phase, each stem node (Alice) stores the transaction in a transaction pool containing only stem transactions: the stempool. The content of the stempool is specific to each node and is non shareable. A stem transaction is removed from the stempool if:

    1. Alice receives it "normally" advertising the transaction as being in fluff mode.
    2. Alice receives a block containing this transaction meaning that the transaction was propagated and included in a block.

3. *Robust propagation.* Privacy enhancements should not put transactions at risk of not propagating. To protect against failures (either malicious or accidental) where a stem node fails to relay a transaction (thereby precluding the fluff phase), each node starts a random timer upon receiving a transaction in stem phase. If the node does not receive any transaction message or block for that transaction before the timer expires, then the node diffuses the transaction normally.

Dandelion stem mode transactions are indicated by a new type of relay message type.

Stem transaction relay message type:
<pre>
Type::StemTransaction;
</pre>

After receiving a stem transaction, the node flips a biased coin to determine whether to propagate it in “stem mode”, or to switch to “fluff mode.” The bias is controlled by a parameter exposed to the configuration file, initially 90% chance of staying in stem mode (meaning the expected stem length would be 10 hops).

Nodes that receives stem transactions are called stem relays. This relay is chosen from among the outgoing (or whitelisted) connections, which prevents an adversary from easily inserting itself into the stem graph. Each node periodically randomly choose its stem relay every 10 minutes.

## Aggregation

Two aggregation scenarios have been proposed.

### Scenario 1: aggregating transaction without lock_height

In this scenario, transactions are aggregated during the stem phase and then broadcasted to the mempool only when the fluff phase happens.

Each node maintains a ```patience``` timer along with a ```max_patience``` value. When a node receives a stem transaction, its ```patience``` timer starts, waiting for more stem transactions to aggregate. Once the ```patience``` timer reachs the ```max_patience``` value, the node flips a coin to decide whether to send as stem or fluff.
In the of a stem transactions, the node starts an embargo timer which is defined in the Considerations part. The node need to track every stem transactions it receives aggregated or not in order to guarantee its propagation and be able to revert to a previous stempool state.

A simulation of this scenario is available [here](simulation.md).

### Scenario 2: aggregating transaction with lock_height

Similar to the previous scenario, except that we aggregate transactions that are locked with ```lock_height```. As of now (7f478d7), transactions with ```lock_height > chain_height``` are rejected by the mempool. In this scenario, such locked transaction would be accepted in the stempool and relayed to other stem relays with the following conditions:

```
if (lock_height <= chain_tip.height && coin_flip <= stem_probability)
```

In this scenario, a patience parameter would still exist to know for how long a peer keep the transaction to its stempool before broadcasting it. Also a ```max_lock_height``` parameter must be defined in order to limit the potential denial of service vector.


## Considerations

The main implementation challenges are: (1) identifying a satisfactory tradeoff between Dandelion’s privacy guarantees and its latency/overhead, and (2) ensuring that privacy cannot be degraded through abuse of existing mechanisms. In particular, the implementation should prevent an attacker from identifying stem nodes without interfering too much with the various existing mechanisms for efficient and DoS-resistant propagation.

* The privacy afforded by Dandelion depends on 3 parameters: the stem probability, the number of outbound peers that can serve as dandelion relay, and the time between re-randomizations of the stem relay. These parameters define a tradeoff between privacy and broadcast latency/processing overhead. Lowering the stem probability harms privacy but helps reduce latency by shortening the mean stem length; based on theory, simulations, and experiments, we have chosen a default of 90%. Reducing the time between each node’s re-randomization of its stem relay reduces the chance of an adversary learning the stem relay for each node, at the expense of increased overhead.
* When receiving a Dandelion stem transaction, we avoid placing that transaction in <code>tracking_adapter</code>. This way, transactions can also travel back “up” the stem in the fluff phase.
* Like ordinary transactions, Dandelion stem transactions are only relayed after being successfully accepted to mempool. This ensures that nodes will never be punished for relaying Dandelion stem transactions.
* If a stem orphan transaction is received, it is added to the <code>orphan</code> pool, and also marked as stem-mode. If the transaction is later accepted to mempool, then it is relayed as a stem transaction or regular transaction (either stem mode or fluff mode, depending on a coin flip).
* If a node receives a child transaction that depends on one or more currently-embargoed Dandelion transactions, then the transaction is also relayed in stem mode, and the embargo timer is set to the maximum of the embargo times of its parents. This helps ensure that parent transactions enter fluff mode before child transactions. Later on, this two transaction will be aggregated in one unique transaction removing the need for the timer.
* Transaction propagation latency should be minimally affected by opting-in to this privacy feature; in particular, a transaction should never be prevented from propagating at all because of Dandelion. The random timer guarantees that the embargo mechanism is temporary, and every transaction is relayed according to the ordinary diffusion mechanism after some maximum (random) delay on the order of 30-60 seconds.

## References
1. (Sigmetrics 2017) Dandelion: Redesigning the Bitcoin Network for Anonymity https://arxiv.org/abs/1701.04439
2. Dandelion++: TBA
3. Dandelion BIP https://github.com/gfanti/bips/blob/master/bip-dandelion.mediawiki
