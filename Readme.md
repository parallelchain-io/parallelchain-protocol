# The ParallelChain Mainnet Protocol

|Revision no.|Version no.|
|---|---|
|0|0.4|

This folder contains the specification for the ParallelChain Mainnet protocol, a protocol which maintains a Byzantine Fault Tolerant, Proof of Stake blockchain with programmability through WebAssembly smart contracts.

ParallelChain Mainnet maintains a singular state called the [world state](World%20State.md), which takes the form of a set of “accounts”. Users modify this state by using an [RPC](RPC.md) API to send digitally-signed instructions called “transactions” into servers called “validators”. These validators batch transactions into blocks, and then link the blocks together to form a sequential history of transactions called a [blockchain](Blockchain.md). A component called the [runtime](Runtime.md) executes transactions, causing deterministic effects on the world state.

Transactions are not free. Users pay [gas](Gas.md) for them to be executed and included in the chain. Transactions are also not only limited to doing a fixed set of operations. Users can write programs called [contracts](Contracts.md) and deploy them to Mainnet, and call them using transactions.

There can be many, many validators, spread across the globe. To make sure that the chain is replicated correctly on all of them, validators execute a consensus algorithm using a library called HotStuff-rs. To execute a consensus algorithm, validators send messages to each other on a peer-to-peer network. Being replicated on many machines makes ParallelChain robust to failures, but it also means we have to be very careful about backwards compatibility when making protocol [updates](Updates.md).

The 7 highlighted items in the description you just read correspond to the 7 different aspects of the ParallelChain Mainnet protocol described in this specification. Each aspect is described in this folder as a separate markdown document. We don’t believe there is one right place to start reading this specification. Just jump to an aspect you are interested in, and start reading.

## If you are reading this document as a PDF

The ParallelChain Mainnet specification is originally written in the form of a set of 7 markdown documents. This PDF combines the 7 markdown documents into a single PDF document, which some readers may prefer, but invalidates some of the inter-document hyperlinks in the text. 

The specification is available in its original format on [GitHub](http://github.com/parallelchain-io/parallelchain-specification).

## Opening an issue

You should open an issue on GitHub if you:
1. Have a feature request / feature idea.
2. Have any questions.
3. Think may have discovered a bug.

Please try to label your issues appropriately.

## Copyright and license

Copyright 2023, ParallelChain Lab.

Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0
