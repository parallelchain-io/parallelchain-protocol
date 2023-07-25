# Peer-to-peer (P2P) networking

|Revision number|
|---|
|0|

ParallelChain replicas maintain a peer-to-peer network (P2P). P2P satisfies *two* high-level requirements:
* **Messaging**: P2P enables replicas to send messages to each other to replicate the state machine, maintain a pool of pending transactions, as well as notify each other of dropped transactions.
* **Peer Discovery**: in order for messages to eventually be received by all of their intended recipients, the P2P network needs to be *connected*. However, forming a connected network topology is challenging, because the number of replicas for any particular ParallelChain blockchain is unbounded and can far exceed the number of transport-layer connections that a computer can feasibly maintain. P2P's peer discovery forms and maintains a connected network topology without requiring every replica to be aware of every other replica.

This chapter describes P2P in four sections:
1. [The first section](#libp2p) briefly describes libp2p, the existing set of protocols using which P2P is built.
2. [The second section](#substreams) describes P2P's substreams layer.
3. [The third section](#messaging) describes how P2P implements messaging.
4. [The fourth section](#peer-discovery) describes how P2P implements peer discovery.

## libp2p


P2P is built using [libp2p](https://docs.libp2p.io/concepts/introduction/overview/). libp2p is a set of networking-related protocols initially developed for [IPFS](https://en.wikipedia.org/wiki/InterPlanetary_File_System) that is intended to help build peer-to-peer systems. 

The libp2p protocols used by P2P fall into two categories:
1. **Substream protocols** define how to resolve domain names into IP addresses, which OSI level-4 transport to use, how to secure connections, and how to multiplex the use of a connection between multiple substreams.
2. **Application protocols** (or just "Protocols") define when to create substreams and what to communicate over the substreams.

libp2p continually evolves, and new versions of libp2p protocols are frequently published. This version of P2P is based on the set of libp2p protocols that is implemented in rust-libp2p versions **v0.51-v0.52**.

## Substreams

P2P substreams are built using four protocols, each implementing a specific functionality listed with before it in the table below:

|Functionality|Protocol|Details|
|---|---|---|
|Domain Name Resolution|[`dns`](https://github.com/libp2p/specs/blob/master/addressing/README.md#ip-and-name-resolution)||
|OSI Level 4 Transport|TCP||
|Authentication and Security|[Noise](https://github.com/libp2p/specs/blob/master/noise/README.md)|Using the XX handshake pattern and the replica's keypair.
|Connection Multiplexing|[yamux](https://github.com/libp2p/specs/blob/master/yamux/README.md) or [mplex](https://github.com/libp2p/specs/blob/master/mplex/README.md)|Negotiated using [multistream-select](https://github.com/libp2p/specs/blob/master/connections/README.md#multistream-select), with yamux preferred.|

## Messaging

TODO:
* Brief description of GossipSub.
* IdentTopic.

### Topics

#### Consensus

TODO:
* `Broadcast`
* `SendTo`

#### Mempool

TODO:
* Research!

#### Dropped Transactions

TODO:
* Research!

## Peer Discovery

TODO:
* Brief description of Kademlia (ask libp2p devs first whether our understanding of "Peer Store"/routing table is correct.
* Brief description of Identify.
* Replication factor.

### Peer discovery

TODO:
* Boot Nodes.
* "Random walk".

