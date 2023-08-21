## About
~~These are some benchmarks taken from [this](https://github.com/Davidson-Souza/utreexo-electrum-server) project.~~

These are some performance benchmarks taken from a few projects I've been working on. They are mostly flamegraphs, for now.

### Flamegraphs
Flamegraphs are simple but powerful measuring tools to understand your program's runtime.
It samples your program every `x` nanoseconds or so, which function is on execution, and the calling stack that lead us to this. A small graph is generated, representing each function as a block, and blocks are stacked on each other to represent calling stacks. A program is shown with it's percentage of CPU time taken from all the program's cpu time, and what other functions/methods it called. At the end, you can have an idea on which functions are taking more cpu time, and what can be improved on.
These graphs are extracted with [this](https://github.com/flamegraph-rs/flamegraph) software, on release mode and running with `--network signet -c config.toml run --use-external-sync`

 - [Flamegraph from January, 4th. 2023](/utreexo-wallet-benchs/utreexo-wallet-flamegraph01042023.svg)
 - [Flamegraph from December, 21st. 2022](/utreexo-wallet-benchs/utreexo-wallet-flamegraph12212022.svg) This one is really cool, because one graph before this (I don't have it here, sorry), about 30% of CPU time was spent on deserializing data from the batch sync. And the reason is simple, I gave a TCP reader to the parser, this is a BAD idea, because when deserializing, the parser will call read for each field, with the specified value. But this is nuts, because each call to read is a syscall!!!!! Now I'm using buffered read, and CPU time consumed by deserialization is negligible.

## Bridge

 - [Flamegraph from November, 20th. 2023](/bridge-flamegraph-20082023.svg) This is my first flamegraph for this software, and I'm happy to see how well [Rustreexo](https://github.com/mit-dci/rustreexo) performs. I dare you to find it in the flamegraph in less than 20 seconds! Annoyingly, the flamegraph is showing a huge `sleep` block, but I'm not sure why. I'm not using any sleep in the code, and I'm not sure if it's a bug in the flamegraph software, or if it's anything related to the async runtime I'm using. I'll investigate this later.