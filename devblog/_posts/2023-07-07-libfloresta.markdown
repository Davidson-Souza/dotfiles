---
layout: post
---
# Libfloresta: Bitcoin full node at your fingertips
## Summary
 - [Introduction](#introduction)
    - [What is Libfloresta?](#what-is-libfloresta)
    - [How does it work?](#how-does-it-work)
    - [What can I do with it?](#what-can-i-do-with-it)
    - [What can't I do with it?](#what-cant-i-do-with-it)
    - [What is the current status?](#what-is-the-current-status)
 - [How can I use it?](#how-can-i-use-it)
    - [Creating a node](#creating-a-node)
    - [Creating a watch-only wallet](#creating-a-watch-only-wallet)
 - [FFI bindings](#ffi-bindings)
 - [Integration with other (Rust) projects](#integration-with-other-rust-projects)
 - [Consensus Code](#consensus-code)
 - [License](#license)
## Introduction
Floresta is a lightweight Bitcoin full node, powered by Utreexo and written in Rust. The primary goal is to build a full node that is easy to run and maintain, while also being fast and resource efficient. During the development, it was clear that this scope could be expanded. Instead of having just one node, how about having a full suite of tools that can be used to interact with the Bitcoin network? This is where Libfloresta comes in. Libfloresta is a library that provides a set of tools that can be used to interact with the Bitcoin network in a lightweight and simple way.

## What is Libfloresta?
Libfloresta provides a series of tools to integrate resource efficient Bitcoin full node functionality into any application. With Libfloresta, developers can easily integrate Bitcoin full node functionality into their applications without worrying about the complexities of running a full node. So any application can give its users full sovereignty over their funds without the need to maintain a separate (and probably overly complicated) server.
As of today, a mobile wallet users needs to either connect to a trusted third-party server, or run a full node on the device at home. The first option is not ideal, as it requires the user to trust the server operator. The second option is also not ideal, as it requires the user to have a device with a lot of storage and processing power, connected all the time with the internet. Libfloresta provides a third option: a lightweight full node that can be run on the mobile device itself, with low footprint and resource usage to a trusted third-party server or run a full node on the device at home.

### How does it work?
Libfloresta is composed of a series of extensible crates, that may be used together or separately. The main crate is `libfloresta`, a meta-crate that provides a simple interface to the other crates. The other crates are:
- `floresta-chain`: a crate that provides chain functionality, such as validating blocks and headers.
- `floresta-wire`: a crate that provides networking functionality, such as connecting to peers and sending/receiving messages.
- `floresta-electrum`: A drop-in replacement for Electrum server, that can be used to serve Electrum clients.
- `floresta-watch-only`: A watch-only wallet, that can be used to keep track of addresses and transactions.
### What can I do with it?
Libfloresta can be used to build a wide variety of applications. Some examples are:
- A mobile wallet that connects to the Bitcoin network directly, without having to trust a third-party server.
- A hardware wallet that is aware of what it is signing, without having to trust the host device.
- A Bitcoin full node that can be run on an older Raspberry Pi, without having to worry about storage and processing power.
- A Web wallet that also runs a full node, even without the user noticing it.
- And many more!

### What can't I do with it?
- Libfloresta is not a drop-in replacement for Bitcoin Core. It is not meant to be used as a backend for a Bitcoin exchange, for example. It is also not meant to be used as a backend for a Bitcoin wallet that is not fully non-custodial. Libfloresta is meant to be used in applications that give the user full sovereignty over their funds, without having to trust a third-party server. If you want to build a Bitcoin wallet that is not fully non-custodial, Libfloresta is not for you.
- Mining with application relying on libfloresta is not advised. Although we try to keep the consensus rules, there may be subtleties that we are not aware of. Mining is a very sensitive subject when it comes to consensus rules, any minor mistake can lead to a chain split. We strongly recommend using Bitcoin Core for mining.

### What is the current status?
Libfloresta is still in early development, but can be used some applications already. The current status is:
- You can create a node, connect to peers, and sync the blockchain.
- We'll validate all blocks and transactions, and keep the UTXO set updated(using Utreexo).
- We have a watch-only wallet, that can keep track of addresses and transactions you are interested in.
- We have a CLI tool that can be used to interact with the node.
- You can use the node to broadcast transactions to the network.

## How can I use it?
Here are some selected examples of how to use Libfloresta. For more examples, check out the [examples](https://github.com/Davidson-Souza/Floresta/blob/master/crates/floresta/examples/) folder.

### Creating a node
In this context, a `Node` means something that can connect with peers and drive our internal chain state machine. You can use alternative implementations for this purpose, in libfloresta we also have a cli-based node, for example. But for now, let's use the p2p based node.
You can find this node in `floresta::wire::node::UtreexoNode`. To create a node, however, we need a `ChainState`. This is one of the core structs in this lib, and is responsible for keeping track of the network state, download new blocks and validate them.
Through this example, we'll see the constant DATA_DIR spinning around, it
is just a directory where we'll store the data. In florestad we use $HOME/.floresta. You can use whatever you want.
```rust
// The chain store is responsible for storing the headers and the accumulator.
let chain_store =
    KvChainStore::new(DATA_DIR.into()).expect("failed to open the blockchain database");

// The actual chainstate. It will keep track of the current state of the accumulator
// and the headers chain. It will also validate new blocks and headers as we receive them.
// The last parameter is the assume valid block. We assume that all blocks before this
// one have valid signatures. This is a performance optimization, as we don't need to validate all
// signatures in the blockchain, just the ones after the assume valid block. We are giving a None value, so we will validate all signatures regardless. We place the chain state in an Arc, so we can share it with other components.
let chain = Arc::new(ChainState::<KvChainStore>::new(
    chain_store,
    Network::Bitcoin,
    None,
));
```

Notice how we use a `KvChainStore` here. This is a chain store that uses a key-value database to store the headers and the accumulator. This is the default chain store, but anything implementing the `ChainStore` trait can be used here.

Now that we have a chain state, we can create a node:
```rust
// Create a new node. It will connect to the Bitcoin network and start downloading the blockchain.
// It will also start a mempool, which will keep track of the current mempool state, this
// particular mempool doesn't store other's transactions, it just keeps track of our own, to
// perform broadcast. We always rebroadcast our own transactions every hour.
// Note that we are using the RunningNode context, which is a state optimized for a node that
// already has the blockchain synced. You don't need to worry about this, because internally
// the node will automatically switch to the IBD context and back once it's finished.
// If you want a node to IBD only, you can use the IBDNode context.
let p2p: UtreexoNode<RunningNode, ChainState<KvChainStore>> = UtreexoNode::new(
    chain.clone(),
    Arc::new(RwLock::new(Mempool::new())),
    Network::Bitcoin, // The network we are connecting to. (Bitcoin, Testnet, Regtest or Signet)
    DATA_DIR.into(),
);
```

If you want to talk to the node, you can take a `handle`, which allows you to request information from the node, like the data associated with a block and a mempool transaction.
You don't need a handle to ask for stuff the chainstate knows, like the current state of the accumulator, or the current height of the chain. You can just use the chainstate directly.
```rust
let handle = p2p.get_handle();
```

To start the node and run, you just have to call the `run` method. This will start the node and connect to peers. It will also start the mempool and the chainstate.
```rust
p2p.run(&Arc::new(RwLock::new(false))).await;
```
Here are two important things to notice:
- We are passing a `Arc<RwLock<bool>>` to the `run` method. This is a stop flag, that will be used to stop the node. If you want to stop the node, just set the value to true. Avoid stopping the node by killing the process, as this may corrupt the database. It is guaranteed that the node will stop gracefully if you set the stop flag to true, after some seconds.
- This function is async, so you need to call it from an async context. If you are not familiar with async rust, you can use the `tokio` or `async-std` runtimes to run it. Here's a full example using `async-std`:

```rust
use async_std::sync::RwLock;
use bitcoin::BlockHash;
use floresta::chain::{pruned_utreexo::BlockchainInterface, ChainState, KvChainStore, Network};
use floresta::wire::mempool::Mempool;
use floresta::wire::node::UtreexoNode;
use floresta::wire::node_context::RunningNode;
use floresta_wire::node_interface::NodeMethods;
use std::str::FromStr;
use std::sync::Arc;

const DATA_DIR: &str = "./data";

#[async_std::main]
async fn main() {

    let chain_store =
        KvChainStore::new(DATA_DIR.into()).expect("failed to open the blockchain database");

    let chain = Arc::new(ChainState::<KvChainStore>::new(
        chain_store,
        Network::Bitcoin,
        None,
    ));

    let p2p: UtreexoNode<RunningNode, ChainState<KvChainStore>> = UtreexoNode::new(
        chain.clone(),
        Arc::new(RwLock::new(Mempool::new())),
        Network::Bitcoin,
        DATA_DIR.into(),
    );

    let handle = p2p.get_handle();
    p2p.run(&Arc::new(RwLock::new(false))).await;
}
```

Alternatively, you can just spawn the node in a new thread, and use the handle to interact with it. This is the recommended way to use the node, as it will allow you to use the node in a sync context.

```rust
...
async_std::spawn(p2p.run(&Arc::new(RwLock::new(false))));

// Here we ask the node to grab the block with the given hash.
let block = handle
    .get_block(
        BlockHash::from_str("000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f")
            .unwrap(),
    )
    .unwrap();
println!("Block: {:?}", block);
```

Just like this, you have a node running. It will connect to the network and start downloading the blockchain. To check if we are synced, we can check the `is_in_ibd` method in the chainstate:

```rust
chain.is_in_idb()   // true if we are in IBD, false otherwise
```

If you want to stop the node, you can just set the stop flag to true:

```rust
*stop_flag.write().await = true;
```

### Creating a watch-only wallet

With `libfloresta` you can build watch-only wallet, specially if your application is built over the Electrum protocol. Here's how to create one

```rust
// This is where we'll store our stuff. We are using one based on Kv here, but can be anything that 
// implements the AddressCacheDatabase trait.
let database = KvDatabase::new("/some_path/").expect("Could not create a database");
// The actual watch-only wallet, for historical reasons, here we call it AddressCache
// you can rename it if you want.
let mut address_cache = AddressCache::new(database)
```

Now that you have a wallet, let's add some addresses for it to follow

```rust
let address = "bc1q33wtsav397tycfrpsjqae5hke6lc9s7z3n975k";
wallet.cache_address(Address::from_str(address)).unwrap();
```

Now you just have to either accept blocks or add unconfirmed transactions.  

If we get a block, just process it as such:

```rust
use bitcoin::Block;

let block: Block = ...;
// We need a block and it's heigh
wallet.block_process(block, height);
```

or, if you broadcast a new transaction, you can just use `cache_mempool_transaction`

```rust
wallet.cache_mempool_transaction(tx);
```

this function will return all coins that conflicts with this tx, like in case of RBF.

#### Fetching data

After you setup a wallet and get some data in it, you can fetch multiple data from it.
Here are a few methods to get started: 
- get_address_utxos: Returns all UTXOs associated with a transaction
- get_cached_transaction: Returns the hex-encoded transaction with this tx_id, if we ever cached it 
- get_position: The position of a transaction inside a block
- get_merkle_proof: The merkle proof for a tx in a block (only for confirmed transactions)
- get_address_balance: The (unconfirmed + confirmed) balance of a address, i.e: amt_in - amt_out
- get_address_history: All transactions associated to a address
- find_unconfirmed: Returns all transactions that are still unconfirmed

## FFI bindings(TODO, actually)

Once we reach a more stable version, we can expose the API to be used in other languages, like Python, JavaScript, Java, etc. The main goal is likely ship something to browsers and mobile, environment that would vastly benefit from Utreexo.

## Integration with other (Rust) projects

This project, in no way pretends to replace the existing rust-bitcoin Bitcoin crates. The main goal here is to bring the power of Utreexo to create compact and simple full nodes that are trivial to integrate and deploy. For wallet tooling and other high-level functions, you'll be better-off with other crates, like `bdk`. A planned feature is to integrate `libfloresta` with other Rust projects, like `rust-lightning` and `bdk`, providing a full stack solution for Bitcoin. This is still a work in progress, though.

## Consensus code
One of the most challenging parts of working with Bitcoin is keeping up with the consensus rules. Given it's nature as a consensus protocol, it's very important to make sure that the implementation is correct. Instead of reimplementing a Script interpreter, we use `rust-bitcoinconsensus` to verify transactions. This is a bind around a shared library that is part of Bitcoin Core. This way, we can be sure that the consensus rules are the same as Bitcoin Core, at least for scripts.

Although tx validation is arguably the hardest part in this process. This integration can be further improved by using `libbitcoinkernel`, that will increase the scope of `libbitcoinconsensus` to outside scripts, but this is still a work in progress.

## License
This project is licensed under the MIT license. See the [LICENSE](https://github.com/Davidson-Souza/Floresta/blob/master/LICENSE) file for more info. To whom it may concern, the author of this project does not care about copies, forks, or any other form of use of this code. You can do whatever you want with it, as long as the no warranty clause is respected.
