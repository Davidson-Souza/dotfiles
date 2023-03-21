# Intoduction

In Utreexo, you need to serialize a proof, in conjunction with the usual data. While sending blocks, the straightforward way is to just serialize proofs with blocks. However, while sending transactions, it can be a big overhead to just serialize a proof for each transaction. There are some ways to do that efficiently.

## Normal transaction propagation flow

In Bitcoin, transactions are propagated between nodes in a best-effort base. When a node receives a transaction, it forwards this transaction to all it's peers. To reduce redundancy, instead of sending a full transaction, nodes send a special message called Inventory, or just Inv. Invs are very simple ways to propagate data you just learned about, like transactions and blocks. In an Inv message, you have a vector of Inv vectors, that can be any of a set of possible types.

If a node learns about some transactions, it sends out an Inv message with all txids as Inv element. Nodes usually don't send out an Inv every time it learns about a new tx, it delays a few seconds to get more data, and sends multiple txs at once.

After getting an Inv, if a node doesn't have it yet, it sends out a getdata request, asking for a transaction. Ideally you only receives the transaction data once. Upon receiving a getdata request, node send a transaction message, with a wire serialized transaction.

## Naive Way

An out-of-the-box way to send proof data is just appending proofs on each transaction in the getdata response, but this has the obvious burden of duplication. One of the goals in utreexo accumulator is creating overlaps of proofs, like in the following tree

```
14
|--------------------------\
12                         13
|-----------\              |-----------\
8            9             10          11
|------\     |------\      |------\    |------\
0       1    2       3     4       5   6       7
```
If we need to prove 1 and 2, the proof is: [0, 9, 13] and [3,  8, 13]. Note that 13 appears on both proofs. Moreover, 8 and 9, appears on each proof, but 9 is computable from 2's proof, and 8 is computable form 1's proof. So to prove both, we only need [0, 13], instead of [0, 8, 9, 13], 50% reduction. In blocks, overlaps are more likely simply because there's  a lot of transactions. Moreover, just sending proofs just kills the purpose of cache.

## A more optimized way
Another way of doing this is sending only targets. Targets are just `u64`, while a full proof have multiple bytes of data, like hashes and leaf preimages, sending only targets gives just a small overhead. This is the serialization of a transaction:


```

+---------------------------------+
| nVersion | nInput  |   input    |
+----------+---------+------------+
| nOut     | output  | nLocktime  |
+---------+-----------------------+

```

This new serialization would add a leaf_number on each input. Intermidiate positions are computable using the targets and number of leaves in the forest. The node would then just look what nodes it doesn't have, then send a new get_position_data, defined as a vector of `u64`s.
The response is a vector of node data, where node data is either a preimage data or just the hash for a leaf or just a node hash for branch nodes. Allowing both preimage and hash fetch of leaves is useful because if we are looking for the target itself, we need the associated data to verify a transaction, but if we are looking for a target's sibling, we don't actually need the preimage.

That way, one can handle overlaps and cache locally and just ask for positions it actually needs.

```
New serialization:
+-----------------------------------+
| nVersion  | nInput  | leaf_number |
+-----------+---------+-------------+
| input     |   nOut  |    output   |
+-----------+---------+-------------+
| nLocktime |
+-----------+

```

### Protocol flow
```
Peer A                                                   Peer B
*Learns a new transaction with `txid  = t`
|                       Inv(t, ...)                          |
|----------------------------------------------------------->|
|                       GetData(t, ...)                      |
|<-----------------------------------------------------------|
|                       Tx                                   |
|----------------------------------------------------------->|  Discovers which positions it needs
|                       GetNodeData(n1, n2, ...)             |
|<-----------------------------------------------------------|
|                       NodeData(h1, h2, ...)                |
|----------------------------------------------------------->|
```
At the end, peer B can recompute proof `P` for this tx and verify against it's current Utreexo state.
This does add a new round trip, but allows some optimizations. For example, if we hold multiple transactions before
sending `GetNodeData` out, we increase the probability of overlaps, further reducing bandwidth consumption.
