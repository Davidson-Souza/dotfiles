---
title: An update on floresta
date: 2024-08-01
author: "Davidson Souza"
tags: [bitcoin, utreexo, node]
layout: post
---

# Floresta: An update on floresta

About a year and a half ago, I've launched floresta, a lightweight bitcoin full node implementation with
utreexo. Since then I've got some inspiring feedback from multiple people, and the project is going through
a major shift in directions, becoming something unique in the bitcoin ecosystem, with a wide potential to
improve security and privacy of users, without requiring dedicated and potentially expensive hardware. This
blog post will go through some of those features and how developers can use those in their applications.

### Reclaiming disk usage

A major barrier when using Bitcoin full nodes is how much disk and I/O it can use. Even if we prune the blockchain, you still need ~15GB of space, and it have to be in a medium that can perform multiple random io operations per second, and sustain a high writing rate. The letter is particularly interesting: some people told me that they tried to run a pruned node in a single board computed using just the SD card for storage. After a few months, the SD card **died**. And this is expected, SD cards are made from cheap, multi-level flash NANDs that degrades really fast in writing heavy workloads. I suspect that many smartphones' internal ROM also suffer from a similar problem, though not as severe as a cheaper SD-card.

This heavy IO load could be solved by caching as much as you can with the volatile memory and only commit to disk occasionally. However, systems with cheap flash memory tends to also have small memory capacity. So there isn't much you can do. Furthermore, the problem will only grow bigger, the UTXO set doesn't seem to stop growing, going almost exponential recently. When I started working on floresta, it was about 6/7GB. Now it's over 10GB.

Luckily, if we are willing to trade some extra bandwidth off, we can reduce the UTXO set to negligible sizes, requiring almost no disk I/O and turning the problem perfectly CPU bound (modern CPUs are great at doing purely logic/arithmetic operations, but will become dead inefficient if it needs to access outside data). Here's a fun experiment: I have a second gen Ryzen 5 and some 24GB DDR4, a somehow fast HDD and fast ethernet connection. If I pull the data directly from another machine running on the same network, florestad takes about 3 seconds to validate a block, without needing to fire-up any any cache. Core on the other hand takes forever if the cache isn't populated, and about 4 seconds to validate if the cache is populated.

During the cache warming, if I `perf stat` them, core is stalling up to 25% of my CPU's backend cycles, while floresta is as little as 3%. The front-end is a little better, with core stalling 6% of the cycles and floresta 2%. Core was also underusing my CPU by a lot, only using about 1.5 billion cycles per second, while floresta uses the full boost performance of this CPU that's about 2.6 billion cycles per second. Those are all reflexes of IO throttling, the HDD isn't fast enough to serve the CPU (I did't profile the HDD usage itself, maybe I should do this in the future). This is not to say that floresta is better than core code-wise (the people working on core are way better engineers than me), this is showing how much this throttling can impact performance, and how a small state could really speed things up if you have a cappy hardware like me.

### Initial Block Download

The process of downloading and verifying every single block on Bitcoin's history is, with no surprise, incredibly time and resource consuming. Skipping it will always involve some tradeoff, but those are arguably better than what most applications in the wild are already making. I've checked the source-code of multiple popular Bitcoin applications, and most of them boils down to "we (the developers) trust this server, and so should you". If we're already trusting the developers, why not trust them to tell us what the state were at a given height? That's how "assume utxo" works. The developers give you a commitment for the UTXO set at a given block, therefore, you only have to download blocks **after** that height.

It turns out that with utreexo, this becomes **waaaaaayyyyy** more efficient. The UTXO set is just a few hashes and a u64. For example, here's the UTXO set for block 855679

```json
"leaf_count": 2588430135,
"root_hashes": [
    "ba0d3416f58c76d2073f1b5def76b78ecb939db84ac30a6ee77a375c848bf8ca",
    "2dc063d9720ee155788967a36aaf1933ed4fa6fa8e80d4894ab80407655f41b2",
    "5caf23d559a8349eb97a75ee909bbd75cdbc3c1043e3454d06a41ccfeed435c6",
    "e65f51785333a2cf4b228e7f9c4294224138d4cf0c76b35094cd38c71b706972",
    "3f601880093912974634a48ffc4f08542529b3120eed4ca45bf297e73c874b7d",
    "a18d7ec9cad420e943844b1db147ce8d87b20f1a5ac0e2baccee91af2194b73b",
    "ccbec6e6e1ed32873c7575aa9178e33ae9b69cfebedbd79dd4a3f6999721ef9f",
    "fecacf67dc97f10a58429cddbfedbbe90de633f91811452ad479b75f21ac4354",
    "ef84006f23bf40658e2421d3d65fda3eea27756fd80c240ad57ec5128668d628",
    "bf253106bb994f9860c3953411bf063ee7f1f0136035dca1d2d5a2b797c83d4f",
    "1ad9d0bc89081c4f5c140caa4f98412a66c1d827b9659880501773690ccf6431",
    "8c024c66fc02cb8f26a350a96e22a9dcadeef15957c8521f52234705218b9a92",
    "d0840de61aec652e1ca7b91d9c0c4c1c81fc37285768456fa6794aaa7234bf9d",
    "5186f5b4e9876d8392300fc043c3e2c125b11d42abd5b0fb382fc94d8371a355",
    "906fbccb60878800e93a3a621fb89633d64e3b660241c11e8f9f32cb4de13173",
    "10b014dae76750e091377acc3300ebe7b3913085af3387096d963816574dfd2c"
  ],

```

This can be incorporated inside a client and you can validate everything from that point onwards. While with assume utxo you may need to wait a little for the UTXO set to load, assume utreexo is instantaneous.

#### Pow fraud proofs

Turns out that there's another way to start you node at an arbitrary height, skipping the history validation. You can use the absence of forks as an indication that there's no violation of the rules. This assumption is only broken if a overwhelming majority joins the colluding side. This not only increases your security threshold from 51% if compared to normal SPV, but also is capable of reacting to any **future** attack, because you'll be validating blocks them. Only a overwhelming majority in **past** can make you accept invalid blocks. I wrote an in-depth description of how this works, you can find it [here](https://blog.dlsouza.lol/2023/09/28/pow-fraud-proof.html).

The important thing is that floresta has an almost functional implementation of this process, that can be tested on signet already. It's not enabled on mainnet right now because it requires patches in other codebase and we are writing tests/making sure it won't break. But we expect to enable this in the next releases for mainnet.

### Libs and FFI support

The inners of floresta (i.e. the chain, p2p, electrum server...) are separated crates that can be used and composed at will by other applications, to reuse only the functionalities that they are interested. Suppose you have a wallet that needs to validate blocks, but you don't need the additional electrum server and watch-only wallet. You can use the `floresta-wire` to connect with peers and pull data from the network, and `floresta-chain` to validate this data and keep the current state.

Here's the [code](https://github.com/Davidson-Souza/Floresta/blob/master/crates/floresta/examples/node.rs) taken from floresta's examples:

```rust

// Create a new chain state, which will store the accumulator and the headers chain.
// It will be stored in the DATA_DIR directory. With this chain state, we don't keep
// the block data after we validated it. This saves a lot of space, but it means that
// we can't serve blocks to other nodes or rescan the blockchain without downloading
// it again.
let chain_store =
    KvChainStore::new(DATA_DIR.into()).expect("failed to open the blockchain database");

// The actual chainstate. It will keep track of the current state of the accumulator
// and the headers chain. It will also validate new blocks and headers as we receive them.
// The last parameter is the assume valid block. We assume that all blocks before this
// one have valid signatures. This is a performance optimization, as we don't need to validate all
// signatures in the blockchain, just the ones after the assume valid block. We are giving a Disabled
// value, so we will validate all signatures regardless.
// We place the chain state in an Arc, so we can share it with other components.
let chain = Arc::new(ChainState::<KvChainStore>::new(
    chain_store,
    Network::Bitcoin,
    AssumeValidArg::Disabled,
));

// Create a new node. It will connect to the Bitcoin network and start downloading the blockchain.
// It will also start a mempool, which will keep track of the current mempool state, this
// particular mempool doesn't store other's transactions, it just keeps track of our own, to
// perform broadcast. We always rebroadcast our own transactions every hour.
// Note that we are using the RunningNode context, which is a state optimized for a node that
// already has the blockchain synced. You don't need to worry about this, because internally
// the node will automatically switch to the IBD context and back once it's finished.
// If you want a node to IBD only, you can use the IBDNode context.
// Finally, we are using the chain state created above, the node will use it to determine
// what blocks and headers to download, and hand them to it to validate.
let config = UtreexoNodeConfig::default();
let p2p: UtreexoNode<RunningNode, Arc<ChainState<KvChainStore>>> = UtreexoNode::new(
    config,
    chain.clone(),
    Arc::new(RwLock::new(Mempool::new())),
    None,
);
// A handle is a simple way to interact with the node. It implements a queue of requests
// that will be processed by the node.
let handle = p2p.get_handle();

let (sender, _receiver) = futures::channel::oneshot::channel();

// Start the node. This will start the IBD process, and will return once the node is synced.
// It will also start the mempool, which will start rebroadcasting our transactions every hour.
// The node will keep running until the process is killed, by setting kill_signal to true. In
// this example, we don't kill the node, so it will keep running forever.
p2p.run(Arc::new(RwLock::new(false)), sender).await;

// That's it! The node is now running, and will keep running until the process is killed.
// You can now use the chain state to query the current state of the accumulator, or the
// mempool to query the current state of the mempool. You may also ask the node to grab some
// blocks or headers for you, or to send a transaction to the network, rescan the blockchain,
// etc. Check the documentation of the node for more information.

// You can't request blocks or headers from the node until it's synced. You can check if it's
// synced by calling the is_in_ibd method.
loop {
    // Wait till the node is synced
    if !chain.is_in_idb() {
        break;
    }
    // Sleep for 10 seconds, and check again
    std::thread::sleep(std::time::Duration::from_secs(10));
}

// Here we ask the node to grab the block with the given hash.
let block = handle
    .get_block(
        BlockHash::from_str("000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f")
            .unwrap(),
    )
    .unwrap();
    println!("Block: {:?}", block);
```

This code will start a node that connects with peers, download blocks from them, validates and notify new blocks to any module that calls [subscribe](https://docs.getfloresta.sh/floresta_chain/pruned_utreexo/chain_state/struct.ChainState.html#method.subscribe). That's it! Any application willing to integrate a local bitcoin client just needs to do this and make sure floresta keeps running on the background. It'll take ~200MB of RAM plus some variable-length caching. On my work machine, it hovers around 600-900 MB of physical RAM usage. But there's always a lot of caching. On my android phone it was around 200-300 while processing thousands of blocks. The CPU and networking load is also very tiny, with a few spikes due to new blocks coming in.

We also have a [Compact Block Filters](https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki) crate to catch-up with historical balance, in case you're using one of the above methods to skip IBD. You can configure the range of blocks we should download filters for, in order to save space. If you want to rescan, just collect all scripts and outpoints you want, call `match_any` and use the node handle to get the returned blocks.

#### Libflorestad

While `libfloresta` is dead simple to use and very powerful, existing applications are still heavily dependant on the `Electrum Protocol`. If you're staring from the grounds-up, you can just use the inner crates and turn enjoy the bells and wissels it gives you. If your application already use electrum to fetch data, you can use `libflorestad`.

Florestad is the daemon built with the daemon, it have the following modules:

 - chain
 - p2p node that connects with peers
 - Compact block filters (optional)
 - Descriptors-ready watch-only wallet
 - Electrum server
 - json-rpc (optional)

Using it couldn't be simpler. Just create a config with some useful settings, like whether to log to a file, whether to compact filters, etc. One important setting is the wallet xpub or output descriptor. We'll use that to derive addresses and keep track of any transaction using this address. If you request the electrum server data about an address that we don't tack, it won't return anything.

After getting a config, just call `Florestad::from_config` and the start method from the created object. You should also remember to stop it before killing the application with the `stop` method. And... that's it!

You can use either the json-rpc or electrum server to pull information. Everything else will be running on the background, and you don't need to worry about it.

#### FFI support

Although the `rust bitcoin` ecosystem is growing, many bitcoin applications are written in other languages. Thankfully, Rust is really easy to call from another language. This is called `Foreign Function Interface` or FFI. `Libflorestad` have some basic bindings for `Swift`, `Python` and `Kotlin`. Those are generated using a tool from Mozilla called ["uniffi"](https://mozilla.github.io/uniffi-rs).

Here's an example on how to call florestad from python:


```python
from floresta import Florestad

daemon = Florestad()
daemon.start()

# do something

#at the end you need to stop the daemon

daemon.stop()
```

The electrum server will be accessible on "127.0.0.1:50001" and the json-rpc on "127.0.0.1:8332" on mainnet (it's the same for core: 8332, 18332, 38332, 48443 for mainnet, testnet3, signet and regtest respectively).

## Second layer protocols and utreexo

One thing I've been thinking about recently is how can floresta handle second-layer protocols, specially those that require an oracle on the UTXO set. Floresta doesn't hold the UTXO set, so we can't ask for arbitrary UTXO. While there are some workarounds to make this possible, it would be easier if those protocols integrated utreexo proofs inside it. Let's take Lightning as an example.

[BOLT-07](https://github.com/lightning/bolts/blob/master/07-routing-gossip.md#the-channel_update-message) defines a handful of messages to tell other nodes about the existence of a channel and the routing information about it. Every channel needs a channel outpoint, that's a onchain UTXO where the multisig between the two parties live. The way this gets referenced is by either the txid and outpoint OR something called short channel id, witch is the block height, tx pos and vout in the format <height>x<pos>x<vout>. If we could just add one extra field "utreexo proof", there wouldn't be any need to have access to the UTXO set. Just verify the UTXO proof, and if it's valid, the channel outpoint exists.

Utreexo proofs needs to be updated after blocks comes by. However, there are efficient algorithms to update large proofs efficiently. Routing nodes could update those proofs once blocks comes by, and if a new node comes and ask about missed gossip, they send the updated UTXO proof. Of course this creates the problem that you can't know if the proof is invalid because a peer is malicious or it is actually a bad commitment. To solve this, you should ban the peer that gave you an invalid proof, and ask other peers for this update.

## Conclusion

Floresta is getting in a stage where it can be used by others to build bitcoin applications with better tradeoffs than the current state of lightweight clients. It offers easy-to-use tools to integrate with and use bitcoin more privately and securely.