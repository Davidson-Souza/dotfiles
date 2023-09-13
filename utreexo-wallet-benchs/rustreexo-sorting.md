While running a benchmark, I noticed that rustreexo was taking quite a long time in calculate_hashes. I did some profiling and found that the culprit was the `sorted_push` function, that just pushes elements into a vector and then sorts it. I didn't think it would be such a big deal, but it turns out that this function is called quite a lot, and takes a lot of time. Here are some experiments I did and the resulting benchmarks.

For reference here is the benchmark before any changes:

```
test accumulator::proof::bench::bench_calculate_hashes ... bench:      83,043 ns/iter (+/- 9,842)
```

See the [latest flamegraph](floresta-flamegraph20182023) for [Floresta](https://github.com/Davidson-Souza/Floresta), you can clearly see that the `sorted_push` function is the bottleneck. This gets worse on mainnet, where there are more transactions and more inputs per transaction.

## Experiment 1: Replace sorted_push with a find and insert

I replaced the sorted_push function with a find and insert function. This function finds the index where the element should be inserted, and then inserts it there. This is faster than sorted_push, but still takes a lot of time. Here are the benchmarks:

```
test accumulator::proof::bench::bench_calculate_hashes ... bench:      67,257 ns/iter (+/- 3,338)
```

Looking at the flamegraph for this benchmark, the `position` function is dominating the entire thing. So we got ~20% speedup, but we sorta moved the problem around. Now the `position` function is the bottleneck.

## Experiment 2: Replace sorted_push with a binary heap

Rust's standard library has a binary heap implementation. I replaced the sorted_push function with a push into a binary heap. This is much faster than the previous two, but still takes a lot of time. Here are the benchmarks:

```
test accumulator::proof::bench::bench_calculate_hashes ... bench:      67,416 ns/iter (+/- 1,281)
```

Very close to the previous experiment. I suspect that trees in general are not a good fit for this problem, because we are constantly inserting and removing elements from the middle of the tree, causing a lot of rebalancing.

## Experiment 3: Similar with experiment 1, but with binary search

Instead of using `position`, which is a linear search, I used a binary search. This is much faster than the previous experiments, in fact, most of the time taken by `calculate_hashes` is now spent on parent hashes. Here are the benchmarks:

```
test accumulator::proof::bench::bench_calculate_hashes ... bench:      57,712 ns/iter (+/- 1,221)
```

I think we have a winner for now. Over 30% speedup from the original benchmark, and the [flamegraph](floresta-flamegraph-13092023) looks much better. The only way this cold be improved is if we could, somehow, avoid copying the entire vector's tail every time we insert an element.