# Peer-to-peer (P2P) networking

|Revision number|
|---|
|0|

ParallelChain replicas maintain a peer-to-peer network (P2P). P2P implements *two* high-level functionalities:
* **Messaging**: P2P enables replicas to send messages to each other to replicate the state machine, maintain a pool of pending transactions, as well as notify each other of dropped transactions.
* **Peer Discovery**: in order for messages to eventually be received by all of their intended recipients, the P2P network needs to be *connected*. However, forming a connected network topology is challenging, because the number of replicas for any particular ParallelChain blockchain is unbounded and can far exceed the number of transport-layer connections that a computer can feasibly maintain. Starting from a small set of "boot nodes", P2P's peer discovery forms and maintains a connected network topology without requiring every replica to be aware of every other replica.

This chapter describes P2P in four sections:
1. [The first section](#libp2p) briefly describes libp2p, the existing set of protocols using which P2P is built.
2. [The second section](#substreams) describes P2P's substreams layer.
3. [The third section](#messaging) describes how P2P implements messaging.
4. [The fourth section](#peer-discovery) describes how P2P implements peer discovery.

## libp2p

P2P is built using [libp2p](https://docs.libp2p.io/concepts/introduction/overview/). libp2p is a set of networking-related protocols initially developed for [IPFS](https://en.wikipedia.org/wiki/InterPlanetary_File_System) that is intended to help build peer-to-peer systems. 

The libp2p protocols used by P2P fall into two categories:
1. **Substream protocols** define how to resolve domain names into IP addresses, which OSI level-4 transport to use, how to secure connections, and how to multiplex the use of a connection between multiple substreams.
2. **Application protocols** define when to create substreams and what to communicate over the substreams.

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

### GossipSub

P2P uses GossipSub for messaging. GossipSub is a topic-based publish-subscribe protocol. A topic is a UTF-8 string, and messages published to a topic is received by peers subscribed to that topic. GossipSub provides robust assurances that a peer subscribed to a topic eventually receives all messages published to that topic. In addition, GossipSub messages are digitally signed by default, and are therefore non-repudiable.

The following subsections describe how P2P configures GossipSub, and then lists the topics that P2P has available for different protocol functionalities, as well as the types in the corresponding message data.

#### Configuration

P2P retains most of the default GossipSub configuration as listed [here](https://docs.rs/libp2p-gossipsub/0.45.0/libp2p_gossipsub/struct.Config.html), but overrides three parameters:

|Overriden parameter|Value|
|---|---|
|Max transmit size|4 KB|
|Allow self origin|true|

### Topics[^1]

[^1]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/7) to create a more logical hierarchy of topics. 

#### State Machine Replication

There are two topics for state machine replication:
* The "consensus" topic: for HotStuff-rs messages that should be received by every replica.
* The individual topic: for HotStuff-rs messages that should be received only by specific replica.

The data type of messages of both topics is `hotstuff_rs::Message`, but each topic will only receive specific variants:

|Topic|Variant
|---|---|
|"consensus"|`ProgressMessage::Proposal`|
|Base64URL encoding of replica's public address|`ProgressMessage::Vote`, or `SyncMessage::SyncRequest`, or `SyncMessage::SyncResponse`|

Every replica must subscribe to the "consensus" topic, as well as the individual topic corresponding to itself.

#### Mempool

After a transaction is submitted to a replica, it is "pending" until included in a block and that block becomes committed. Pending transactions are initially held in a component called the mempool.

The "mempool" topic is used to replicate a pending transaction across all mempools. This reduces the time a transaction has to wait to become included in a block. When a replica receives a transaction through the `submit_transaction` RPC, it broadcasts it to all other replicas using this topic.

|Topic|Data type|
|---|---|
|"mempool"|Transaction|

Every replica must subscribe to the "mempool" topic.

#### Transaction Drop Notification[^2]

A transaction may cease being pending at a replica while it hasn't been included in a block that is then committed. These transactions are said to be "dropped". A transaction may be dropped for a variety of reasons.

The "droppedTx" topic is used to notify clients and other software when a transaction is dropped. Replicas do not have to subscribe to "droppedTx".

|Topic|Data type|
|---|---|
|"droppedTx"|Transaction Drop Notification|

**Transaction Drop Notification** is an enum with two variants:

|Variant|Name|Fields|
|---|---|---|
|0|Dropped by Mempool|<ul><li>transaction (`Transaction`)</li><li>reason (`TransactionDropReason`)</li></ul>|
|1|Dropped by Executor|<ul><li>transaction (`CryptoHash`)</li><li>reason (`TransactionDropReason`)</li></ul>|

**Transaction drop reason** is a type alias for `u64`. It has three possible values:

|Name|Value|
|---|---|
|Invalid|515|
|Nonce too low|516|
|Nonce inaccessible|517|

[^2]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/13) to fully flesh out this functionality and move it to the RPC.

## Peer Discovery

### Kademlia

Kademlia is a Distributed Hash Table (DHT) with a network topology that has desirable properties for peer-to-peer networks with large numbers of participants. P2P *disables* Kademlia's storage functionality and uses it solely for Peer Discovery.

Kademlia maintains a table (a “routing table”) of peer address information in each peer. Replicas should initialize the routing table by adding a set of boot nodes and running the “bootstrap” operation. Replicas should then continuously refresh the routing table by periodically triggering `FIND_NODE` on a randomly chosen key. 

Every time Kademlia opens a new connection to a peer, GossipSub is notified and considers opening a stream to that peer for itself, eventually creating a connected topology of GossipSub peers.

#### Configuration

As with GossipSub, P2P retains most of Kademlia's default configuration (as listed [here](https://docs.rs/libp2p/latest/libp2p/kad/struct.KademliaConfig.html#method.set_replication_factor)), and overrides two parameters:

|Overriden configuration|Value|
|---|---|
|Protocol names|"/pchain_p2p/v1.0.0"[^3]|
|Record filtering|Filter both. Discard all incoming records.|

[^3]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/7) to make protocol name a parameter that is individual to each ParallelChain protocol P2P network instance.

