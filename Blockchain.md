# Blockchain 

|Revision no.|
|---|
|0|

The ParallelChain blockchain is a sequence of blocks, which in turn are sequences of transactions.

Executing a transaction creates a receipt, a compact description of what happened in a transaction. When a validator becomes the leader, it uses the runtime to execute pending transactions, and then packs admissible transactions and their receipts into a block. It then passes the block to the consensus provider, which works to replicate the block in all replicas.

This document specifies the block-level features of this flow. It is organized into three sections: 
1. The [first section](#blockchain-data-types) specifies the data types that are included in the blockchain, this includes chiefly transaction and block. 
2. The [second section](#replication) specifies how the ParallelChain protocol uses HotStuff-rs to replicate the blockchain. 
3. Throughout this document, we define several constants that we  generally refer to as "limits", things like the maximum size of blocks, or the maximum time replicas wait for a leader to propose a block before listening to the next leader. These limits are decided through semi-standardized experiments on real hardware. These are described in the [third and final section](#deciding-limits).


## Blockchain data types

### Block

A **block** is a structure composed of a fixed-size header and variable-size data (transactions and receipts). It has fields:

|Field|Type|Description|
|---|---|---|
|Header|`BlockHeader`|[Next section](#block-headers).|
|Transactions|`Vec<Transaction>`|A list of transactions.|
|Receipt|`Vec<Receipt>`|A list of receipts. One for each transaction in the block, and in the same order.|

#### Epoch

An epoch is a sequence of blocks, and therefore indirectly a length of time, through which membership of the validator set is constant. This is analogous to the notion of a “term” in a democratic government, during which the membership of a parliament or a legislative body is fixed. All epochs are $B_{epochlen}$ long, except the first (0th) epoch, which is one block longer ($[0, B_{epochlen}]$).

|Name|Value|Description|
|---|---|---|
|$B_{epochlen}$|8,640|Blocks at heights that are a multiple of this (except the 0th block) are epoch boundary blocks.|

Blocks whose height is a multiple of $B_{epochlen}$ (except the 0th block) are *epoch boundary blocks*. 

Epoch boundary blocks are special in that they contain only one transaction, signed by the block's proposer. This transaction in turn only has one command: [next epoch](Runtime.md#next-epoch). Epoch boundary blocks cause the validator set to be changed.

### Block header

A **block header** summarizes the data of a block, as well as contain metadata used in consensus and gives context about the block's creation and its position in the chain:

|Field|Type|Description|
|---|---|---|
|Hash|`CryptoHash`|The SHA256 hash over (Height, Justify, Data Hash).|
|Height|`u64`|The number of justify-links between this block and the genesis block 0. 0 in the genesis block.|
|Justify|`QuorumCertificate`|A Quorum Certificate that points to the block's parent.|
|Chain ID|`u64`|A number unique to this particular ParallelChain Mainnet-based chain's history. This prevents blocks from one chain being confused as being for another chain. For Mainnet, this is 0.|
|Proposer|`PublicAddress`|The public address of the validator that is the leader of the view this block was proposed in.|
|Timestamp|`u64`|A unix timestamp. This must be in the range `[prev_block.timestamp, local_time + 15]`.|
|Base Fee Per Gas|`u64`|The (inclusive) minimum number of grays that a transaction in must pay for every [gas](Runtime.md#gas-used) used to be included in this block. How this value is decided is specified in [base fee formula](#base-fee-formula).|
|Gas used|`u64`|The gas used to store and execute all of the block's transactions. This is exactly the sum of: <ol><li>The $G_{txincl}$ of every transaction in the block.</li> <li>The value in the gas used field of every command receipt in the block.|
|Transactions Hash</li></ol>|`CryptoHash`|The root hash of the SHA256 binary merkle tree over the blocks' transactions, constructed using [rs_merkle](https://docs.rs/rs_merkle/latest/rs_merkle/). Each leaf is a transactions' hash, and each transaction are placed in the tree in the order they appear in the block.|
|Receipts Hash|`CryptoHash`|The root hash of the SHA256 binary merkle tree over the blocks' receipts. This is similar to transactions hash, but each leaf is the SHA256 hash of a receipt's bytes encoding.|
|State Hash|`CryptoHash`|The root hash of the [world state](World%20State.md) after executing this block.
|Logs bloom|`[u8; 256]`|A bloom filter generated over all logs in the block's receipts. This starts as a 2048-bits ($2^11$ bits) array of all zeros, then is populated using the following procedure: for every log, SHA256-hash its topic, take the first 11 bits of each of the three least-significant pairs of bytes in the digest, and then use each of these 11 bits to index into a bit in the log bloom and set it to 1.|

The amount of the network's computation and storage resources that a block can consume is limited by a cap on gas used:
|Name|Value|Description|
|---|---|---|
|$B_{maxgasused}$|250,000,000|The (inclusive) maximum value of the gas used field in a block's header.|

#### Base fee formula

|Name|Value|Description|
|---|---|---|
|$B_{minbasefee}$|8|The minimum allowed value in a block's base fee per gas field.|
|$B_{tgtgasused}$|$B_{maxgasused}/2$|The target gas used in a block.|

The base fee per gas of a block is a function of the parent's base fee per gas, and the parent's gas used. It increases if the parent's gas used is greater than $B_{tgtgasused}$, and decreases if it is smaller, down to a minimum of $B_{minbasefee}$. 

The specific formula for a block's base fee per gas is:

$$
B_{basefee}(pbf,pgu) =  max\left(B_{minbasefee},\ pbf + B_{basefeedelta}(pbf,pgu)\right)
$$

where $pbf$ is the parent's base fee per gas, and $pgu$ is the parent's gas used. $B_{basefeedelta}$ is:

$$
B_{basefeedelta}(pbf, pgu) = max\left(1,\ \frac{pbf \times \left(pgu - B_{tgtgasused}\right)}{8 \times B_{tgtgasused}}\right) 
$$

The intended result of the base fee formula is to use pricing to adjust demand such that blocks are half-full in the steady state. 

This mechanism is nearly identical to [Ethereum's](https://github.com/paradigmxyz/reth/blob/8987f4e8be60b92d596acc21a2b27ecc4ede9f1a/crates/consensus/common/src/validation.rs#L245) in all respects except for two:
1. $B_{basefeedelta}$ is not piecewise. This is intentional.
2. The minimum base fee per gas is 8 instead of 7. This difference is a design oversight, and one that we intend to rectify in the next protocol major version.

### Transaction

A **transaction** is a digitally signed instruction by an identified party (the "signer") that authorizes the blockchain to execute a sequence of [commands](Runtime.md):

|Field|Type|Description|
|---|---|---|
|Signer|`PublicAddress`|The public address of the [external account](World%20State.md) which signed this transaction.|
|Nonce|`u64`|The number of transactions signed by the signer that have been included in the blockchain before this transaction.|
|Commands|`Vec<Command>`|The sequence of commands that this transaction instructs the runtime to execute.|
|Gas Limit|`u64`|The maximum number of gas units that should be used to execute this transaction before halting.|
|Max Base Fee per Gas|`u64`|The maximum number of grays per gas used that the signer is willing to burn (80%) or give to the treasury account (20%).|
|Priority Fee per Gas|`u64`|The maximum number of grays per gas used that the signer is willing to give to the proposer of the block this transaction is included in.|
|Signature|`Signature`|An Ed25519 signature using the signer's private key over this transaction, but with the signature and hash fields all zeroed.| 
|Hash|`CryptoHash`|The SHA256 hash of the populated signature field.|

### Receipt

**Receipt** is a type alias of `Vec<CommandReceipt>`, and is a compact summary of what happened during the execution of a transaction.

More precisely, a receipt is a sequence of **command receipts**, each of which describe what happened during the transactions' commands:

|Field|Type|Description|
|---|---|---|
|Exit code|`ExitCode`|[An enum](#exitcode) that informs how the command exited.|
|Gas used|`u64`|The amount of gas used in executing the command.|
|Return value|`Vec<u8>`|Generic data with meaning that varies according to the variant of command that created the receipt.|
|Log|`Vec<Log>`|[Logs](#log) created during the execution of the command.|

#### Exit code 

An **exit code** is an enum included in a command receipt that informs how a command exited. A command can exit in one of three ways:

|Variant|Description|
|---|---|
|Operation successful|The command successfully did what it is supposed to do.|
|Operation failed|The command failed to do what it is supposed to do.|
|Gas exhausted|Execution *halted* in the middle of the command because the gas limit was hit.|

The above descriptions of "operation successful" and "operation failed" are intentionally vague. What "operation successful" and "operation failed" means depends on the variant of the command. The runtime document has a [table](Runtime.md#commands) that should help with interpreting exit codes, as well as return values, in the context of command variants.

### Log

Besides return values, a **log** is another way commands can put generic data about its execution onto the chain. Logs have the benefits over return values that their topics are combined into the block's [log bloom](#block-header), and that a command receipt can have multiple logs.

A log is a structure with fields:

|Field|Type|Description|
|---|---|---|
|Topic|`Vec<u8>`|Generic data that is combined into the block's log bloom.|
|Value|`Vec<u8>`|Generic data.|

## Replication

Safe replication of the ParallelChain blockchain is enabled by two crucial aspects:
1. [Delegated Proof of Stake (DPoS)](#delegated-proof-of-stake) defines the "admissions requirements" on parties who want to become validators, and establishes incentive structures that encourages them to behave honestly.
2. [Byzantine Fault Tolerant State Machine Replication](#state-machine-replication) (BFT SMR) defines the algorithm that validators execute together to grow a consistent blockchain, even in the face of faults. 

The following subsections discuss the two aspects in turn.

### Delegated Proof of Stake

ParallelChain is a Delegated Proof of Stake (DPoS) blockchain protocol. This means that the set of *validators* that can propose and vote for blocks at any given moment is determined by how many tokens they have *deposited* and *staked*. In particular, the protocol aims through the design of [staking commands](Runtime.md#staking-commands) for the $B_{vscap}$ largest pools by total stake to become the validators in a given [epoch](#epoch). 

A **validator** is a pool whose operator can propose and vote for blocks in a given epoch.

A **pool** is a collection of stakes that competes with other pools in total stake to become a validator. Every pool has an identified *operator*, the account which will propose and vote for blocks on behalf of the pool if it becomes a validator. An account can be the operator of at most a single pool.

A **deposit** is some balance that an owner has temporarily relinquished with the hope that it will become part of a specific pool. A **stake** is some balance that has actually become part of a pool. Creating a stake is a two-step process, one deposits a balance, and then stake the deposit.

There are two kinds of stake: An **operator stake** is a stake in a pool whose operator is the owner of the stake itself. A **delegated stake** is the converse: a stake in a pool whose operator is not the owner of the stake.

Similarly to how only the top $B_{vscap}$ largest pools become part of the validator set, a pool can contain a maximum of $B_{poolcap}$ *delegated* stakes. Not similarly, however, is that stakes are either part of a pool, or it doesn’t exist: a stake deposit command that tries to create a stake that is smaller that the smallest stake currently in the pool will fail, and a stake that is  displaced by a larger stake ceases to exist (but their balance remains in its associated deposit). This is unlike how pools can exist but not be part of a validator set.

|Name|Value|Description|
|---|---|---|
|$B_{vscap}$|63|The maximum size of the validator set.|
|$B_{poolcap}$|127|The maximum number of delegated stakes in a pool.|

Users do the basic operations of DPoS (e.g., creating, staking, unstaking, and withdrawing a deposit) by creating transactions that include [staking commands](Runtime.md#staking-commands). 

## State Machine Replication 

The ParallelChain protocol depends on release 0.2.0 of [HotStuff-rs](https://github.com/parallelchain-io/hotstuff_rs) for state machine replication. HotStuff-rs is an implementation of the HotStuff BFT SMR algorithm.

HotStuff-rs interacts with replica implementations of the ParallelChain protocol through two simple traits: 
- [App](#app), which encapsulates logic for producing and validating blocks, and
- [Pacemaker](#pacemaker), which dictates how the logical clocks ("views") of replicas are synchronized, and which validator gets to propose a block at a given view. 

### App

The app trait requires that replica implementations implement two functions: *produce block*, and *validate block*. In the former, leaders produce a block that satisfies the ParallelChain Mainnet specification. In the latter, replicas validate whether a proposed block satisfies the ParallelChain Mainnet specification.

Calls to produce an [epoch boundary blocks](#epoch) additionally return the changes to the validator set between the current epoch and the next epoch.

### Pacemaker

Generally speaking, state machine replication algorithms for the partially synchronous network model (in which network latency may spike during exceptional periods) synchronize the actions of participating replicas by ensuring that they eventually agree on the same "view". A view is a number used to decide which validator in the validator set becomes a *leader* and gets the exclusive right to propose a block during a given period of time. 

[Pacemakers](https://docs.rs/hotstuff_rs/latest/hotstuff_rs/pacemaker/index.html) ensure view synchronization by specifying functions that decide:
- how long a replica should stay in a view (*view timeout*) and
- who should be the leader of a given view (*view leader*).

The ParallelChain protocol uses HotStuff-rs' [default pacemaker](https://docs.rs/hotstuff_rs/latest/hotstuff_rs/pacemaker/struct.DefaultPacemaker.html) with $B_{tbt}$ as the value for minimum view timeout. 

|Name|Value|Description|
|---|---|---|
|$B_{tbt}$|10|Target block time in seconds.|
|$B_{netlatency}$|2|Expected worst-case network latency.|

$B_{tbt}$ is the number of seconds that a replica spends in a given view to process a block proposed by the designated leader. This includes both the time waiting for a proposal, and the executing the block to validate it.

Since $B_{tbt}$ needs to account for executing the block twice (once at the leader to produce it, and then at replicas to validate it) and also network latency, a leader should spend at most $\frac{B_{tbt}}{2} - B_{netlatency}$ seconds producing a block so that replicas will vote for it.

## Deciding limits

The block gas limit $B_{maxgasused}$ was decided by running a standardized benchmark on a Linux-based machine with Fullnode installed that we consider to be a conservative approximation of a fully deployed ParallelChain Mainnet network.

Some information about the machine:

|Attribute|Value|
|---|---|
|Hosting provider|AWS EC2|
|Instance|c5.4xlarge|

The benchmark involves a minimal contract implementation of a fungible token (FT) that when called, simulates a token transfer. 

The goal was to choose a block gas limit that allows us to produce a block roughly every 10 seconds, and allows for a usable but conservative number of call-FT commands and deploy-FT commands to be successfully completed in a block without causing view timeouts.

We do this by pre-generating a large number of unique call-FT transactions (transactions with only one command: call-FT) and distributing them to the 10 validators in the testing cluster. 

When a validator becomes a leader, it executes transactions and produces a block. The validators are configured to keep executing transactions until a block’s gas limit is reached.

Then, we progressively increase the gas limit until the execution time increases to the point that view timeouts begin occurring. 

The experimental result was that view timeouts begin occurring when $B_{maxgasused}$ is set to higher than 250 billion, and that with the gas limit at this value, a block could at most include 150 call-FT transactions, and 2-3 deploy-FT transactions. We deem these figures acceptable.
