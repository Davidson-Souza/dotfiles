# Rustreexo: A Rust implementation of Utreexo in pure Rust

## Summary
- [Rustreexo: A Rust implementation of Utreexo in pure Rust](#rustreexo-a-rust-implementation-of-utreexo-in-pure-rust)
    - [Summary](#summary)
    - [Introduction](#introduction)
    - [Rustreexo](#rustreexo)
        - [Stump](#stump)
        - [Pollard](#pollard)
    - [Conclusion](#conclusion)
    - [Footnotes](#footnotes)

## Introduction

Utreexo is a hash-based, accumulator-based, authenticated data structure that can be used to replace the current UTXO set in Bitcoin. It was first proposed by Tadge Dryja in 2019. The main idea is to use a forest of Merkle trees to store the UTXO set, and use a proof to prove that a UTXO is in the UTXO set. The proof is a path from the leaf node to the root node of the Merkle tree. The size of the proof is O(logN), where N is the number of UTXOs. The UTXO set can be updated by adding or deleting a UTXO. The update time is O(logN). The storage space of the UTXO set is O(N). The storage space of the proof is O(logN).

## Rustreexo

Rustreexo is a Rust implementation of Utreexo in pure Rust. It implements the necessary functions of Utreexo, including adding, deleting, and verifying UTXOs. Although the algorithm is the same, Utreexo allows different representations of the accumulator data. Wallet and lightweight clients won't need to store anything
other than the roots, while special nodes, know as Bridge Nodes, must keep the entire set, in order to generate proofs for arbitrary UTXOs. It has a very minimal set of dependencies, and is designed to be easy to use and integrate into other projects.

Rustreexo implements both representations. A lightweigt client can use the `Stump` struct, which only stores the roots. A Bridge Node can use the `Pollard` struct, which stores the entire set.

## Stump

As noted above, a Stump is a simple and lightweight data structure that only stores the roots of the Merkle tree. It can be used by a lightweight client to verify the existence of a UTXO. Using a Stump, a user can also prove and update proofs for elements it has interest in, like the wallet's UTXO.

A simple example of how to use a Stump is shown below(this example is borrowed from [here](https://github.com/mit-dci/rustreexo/blob/main/examples/simple-stump-update.rs)):

We first need some UTXOs to add to the Stump. In Bitcoin, these would be the UTXOs created in a block. If we assume this is the very first block, then the Stump is empty, and we can just add the UTXOs to it. Assuming a coinbase with two outputs, we would have the following UTXOs. Note that we are using hashes, with type `NodeHash`, this type is used everywhere in Rustreexo, and is just a wrapper around a 32-byte array with some useful methods.

To obtain this hash, you Sha512_256 the following data:
```
    Block hash in which the UTXO was created.
    Transaction id of the transaction that created the UTXO.
    Index of the output in the transaction as little-endian u32.
    Header code of the UTXO¹ as little-endian u32.
    Serialized UTXO in wire format.
```

For this example, we will give the hashes directly, but in a real implementation, you would need to calculate them.
```rust
let utxos = vec![
    NodeHash::from_str("b151a956139bb821d4effa34ea95c17560e0135d1e4661fc23cedc3af49dac42")
        .unwrap(),
    NodeHash::from_str("d3bd63d53c5a70050a28612a2f4b2019f40951a653ae70736d93745efb1124fa")
        .unwrap(),
];
```

Create a new Stump, and add the utxos to it. Notice how we don't use the full return here, but only the Stump. We will see what the second argument is used for later.

```rust
let s = Stump::new()
    .modify(&utxos, &[], &Proof::default())
    .unwrap()
    .0;
```
The modify method is a pure-function, that takes as arguments two slice references and one proof. Being a pure function means that it doesn't modify the Stump, but instead returns a new one. The first slice is the UTXOs to add, the second is the UTXOs to remove, and the proof is a proof that the UTXOs to remove are in the Stump. In this case, we are adding all the UTXOs, and removing none, so we pass an empty slice for the second argument, and a default proof for the third. The default proof is a proof that the empty slice is in the Stump, which is always true.

Now we can verify that the UTXOs are in the Stump. We can do this by creating a proof, and then verifying it. The proof is a path from the leaf node to the root node of the Merkle tree. The size of the proof is O(logN), where N is the number of UTXOs. For our simple example with only two UTXOs, the proof for Utxo 0 is just
the position 1.

```rust
// Create a proof that the first utxo is in the Stump.
let proof = Proof::new(vec![0], vec![utxos[1]]);
// Verify the proof.
assert_eq!(proof.verify(&[utxos[0]], &s), Ok(true));
```

Now we want to update the Stump, by removing the first UTXO, and adding a new one. This would be in case we received a new block with a transaction spending the first UTXO, and creating a new one.

```rust
let new_utxo =
    NodeHash::from_str("d3bd63d53c5a70050a28612a2f4b2019f40951a653ae70736d93745efb1124fa")
        .unwrap();
let s = s.modify(&[new_utxo], &[utxos[0]], &proof).unwrap().0;
```

Now we are passing the proof we created earlier, removing the first utxo. We also provide 0's hash so it can be removed. We are adding one new UTXO, so the UTXO set
still has two UTXOs, but now we have one STXO. If we try to verify the proof for the first UTXO, it will fail, because it is no longer in the Stump.

```rust
assert_eq!(proof.verify(&[utxos[0]], &s), Ok(false));
```

Now, suppose we are a wallet, and the second UTXO is ours. We want to generate and maintain the proof for it, so we can spend it. We can do this by creating a new proof, and then updating the Stump with it.

Rustreexo provides a method to generate a proof for a given UTXO. This method takes as argument the position of the UTXO in the Stump, and returns a proof for it. In this case, the position is 1, because it is the second UTXO in the Stump. Remember that extra return from Stump::Update? This is where we use it. The second return is like a diff of the Stump, and we can use it to update proofs.

First, lets create a proof for the second UTXO.

```rust
let (s, update) = Stump::new()
    .modify(&utxos, &[], &Proof::default())
    .unwrap();

let (p, cached_hashes) = p
    .update(vec![], utxos.clone(), vec![], vec![0, 1], update_data)
    .unwrap();
```
Update, is a method to incrementally build and update proofs over the accumulator's data.  This proof was initially empty, but we can instruct this function to remember some UTXOs, given their positions in the list of UTXOs we added to the accumulator. In this example, we ask it to cache 0 and 1. Cached hashes are the hashes we care about, after this update, it'll be the hashes of 0 and 1. Add hashes are the newly created UTXOs hashes, block targets are the STXOs being spent. update_data is the data we got from the accumulator update, and contains multiple intermediate data we'll need.

Now we can verify it against the Stump.

```rust
assert_eq!(p.verify(&cached_hashes, &s), Ok(true));
```

Now, suppose we receive a new block, with a transaction spending the second UTXO, and creating a new one. We can update the Stump, and the proof for the second UTXO, by using the update method.

```rust
let new_utxo =
    NodeHash::from_str("d3bd63d53c5a70050a28612a2f4b2019f40951a653ae70736d93745efb1124fa")
        .unwrap();
let (s, update) = s.modify(&[new_utxo], &[utxos[1]], &p).unwrap();
let (p, cached_hashes) = p
    .update(cached_hashes, vec![new_utxo], vec![utxos[1]], vec![], update_data)
    .unwrap();
```

after that, our proof is still valid.

```rust
assert_eq!(p.verify(&cached_hashes, &s), Ok(true));
```

If you have multiple UTXOs in a proof, and needs only a subset of them, you can use the `Proof::get_proof_subset` method to create a new proof for the subset of UTXOs you need. This method takes as argument the positions of the UTXOs you want in the new proof, and returns a new proof for them.

```rust
let p = p.get_proof_subset(vec![0]).unwrap();
```

Keep in mind that having multiple proofs is redundant, and you can always use the original proof to verify any subset of UTXOs. Updating multiple proofs is probably less efficient than updating a single one, but it is still possible. A normal wallet would probably only need one proof of a handful of UTXOs, and update it as needed.

# Pollard

Pollard is another way to represent the Utreexo State, but keeping track of the entire (or parts of the) tree, instead of just the roots, allowing it to prove arbitrary UTXOs. As such, it is more expensive, requiring O(N + (2 * N - 1)) space, where N is the number of UTXOs.

It is called Pollard because it allows for pruning some of the leaves, and only holding a subset of them. Bridge nodes may use this to speed-up proving time, since we know by a fact that many UTXOs will likely never be spent, but removing them from the UTXO set is a soft-fork, and would require a lot of coordination.
As of today, rustreexo does not support pruning, but it is planned for the future.

Here's how to use it. First, create a new Pollard.

```rust
let mut p = Pollard::new();
```

Now, we can add UTXOs to it. Similarly to the Stump, we can add multiple UTXOs at once, and we can also remove UTXOs. The difference is that for deletion, we don't need to prove anything.

```rust
p.modify(&utxos, &[]).unwrap();
```

Create a proof for a given UTXO. This method takes the hash of a leaf, and returns a proof for it, along with the sorted target hashes.

```rust
let (proof, hashes) = p.prove(&[utxos[0]]).unwrap();
```

Now we can verify the proof against the Accumulator.

```rust
assert_eq!(proof.verify(&hashes, &p), Ok(true));
```

That's it. If we have a valid proof for a UTXO, and a new block comes in, we may either update the proof, or call Prove again.

# Conclusion

Utreexo is a important tool to scale Bitcoin, and rustreexo is a efficient and easy to use implementation of it. It is still in early stages, and there are many things to be done, but it is already usable, and we are working on improving it.

If are looking for more complete examples, check out the examples in the [repository](https://github.com/mit-dci/rustreexo/blob/main/examples/). If you have any questions, feel free to open an issue, or ping us on IRC #utreexo on Libera.
# Footnotes

[1] Header code is a compact way to store the height of a UTXO and whether it is a coinbase, and is defined as
```
header_code: u32 = if transaction.is_coinbase() {
    (block_height << 1 ) | 1
} else {
    block_height << 1
};
```
