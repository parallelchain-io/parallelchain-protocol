# World State

|Revision no.|
|---|
|0|

Following in Ethereum, we call the singular user-visible state that the ParallelChain protocol maintains the "world state". The
world state is a set of key-value tuples representing the state of every *account*, stored inside a [paritytech/trie-db](https://docs.rs/trie-db/latest/trie_db/) Merkle Patricia Trie (MPT).

This document specifies the contents of the world state, and is organized into three sections:
1. The [first section](#accounts) lists the state that is stored in a general account.
2. The [second section](#network-account-storage) gives a high-level overview of the state that is stored in the storage of a special account called the "network account", which is used to support [delegated proof of stake](Blockchain.md#delegated-proof-of-stake).
3. The [third section](#network-account-storage-data-types) specifies the data structures used in the network account storage.

## Accounts

An account is an agent associated with some state, and which can trigger state changes in specific ways. 

There are two main kinds of account, and they differ in how they trigger state changes: 
1. *External accounts*, which trigger state changes by digitally signing transactions, and
2. *Contract accounts*, which trigger state changes when [called](Runtime.md#call) according to the logic in its code.

Each account is uniquely identified by a 32-byte sequence called a *public address*. An external account's public address can be any Ed25519 public key, while a contract account's public address is the SHA256 of the concatenation of the signer and nonce fields of the transaction which [deployed](Runtime.md#deploy) it.

Each account has 5 fields, each stored in the world state at a key formed by concatenating the account's address with a specific 1-byte key suffix.

|Field|Type|Suffix|Description|
|---|---|---|---|
|Nonce|`u64`|0|In an external account, the number of transations signed by the account so far in the blockchain history. In a contract account, empty.|
|Balance|`u64`|1|The number of XPLL ("tokens") in the account, in grays.|
|Contract|`Vec<u8>`|2|In a contract account, a [valid contract](Contracts.md#contract-validity). In an external account, empty.|
|CBI version|`u16`|3|In a contract account, the major version of the CBI that the contract targets.|
|Storage hash|`CryptoHash`|4|In contract accounts and the network account, the root hash of their individual storage trie. In an external account, empty.|

All token balances and amounts related to balances (e.g. stake powers) are stored in grays:

|Name|Value|Description|
|---|---|---|
|$W_{xplltogray}$|$10^8$|Conversion factor between XPLL and gray. 1 XPLL = $W_{xplltogray}$ grays.|

## Network account storage

Besides state that are tied to particular external or contract accounts, the world state also keeps track of state that have significance to the entire network. This state is stored in the storage trie of a special account called the network account, which has a public address of `[0u8; 32]`. The main purpose of this state is to implement delegated proof of stake (DPoS)

This state consists of 6 fields:

|Field|Type|Suffix|Description|
|---|---|---|---|
|Previous validator set|`IndexHeap<Pool, B_vscap>`|0|The validator set in the previous epoch.|
|Current validator set|`IndexHeap<Pool, B_vscap>`|1|The validator set in the current epoch.|
|Next validator set|`IndexHeap<PoolKey, B_vscap>`|2|The $B_{vscap}$ largest pools that are competing to become the next validator set.|
|Pools|`PublicAddress -> Pool`|3|All pools that are accepting stake and competing to become part of the next validator set, indexed by the operator of the pool.|
|Deposits|`PublicAddress -> Deposit`|4|All deposits.|
|Current epoch|`u64`|5|The current epoch.|

The above fields uses a few types that we haven’t described: `L -> R`, `Pool`, `Deposit`, and `IndexHeap<E, S>`

`L -> R` is very generic and denotes a mapping between a key of type L, and a value of type R. This mapping is created by storing the R value at the path of the mapping type suffixed with the L value.  

The other types are more complex and have more precise semantics, and are specified in the next section.

## Network account storage data types

### Pool

A pool is described in 5 fields:

|Field|Type|Suffix|Description|
|---|---|---|---|
|Operator|`PublicAddress`|0|The operator of the pool.|
|Power|`u64`|1|The sum of the powers' of the pool's operator stake and delegated stakes.|
|Commission rate|`u8`|2|the percentage (0-100%) of the epoch’s issuance rewarded to the pool that will go towards the operator’s stake (or their balance, if the operator did not stake to itself).|
|Operator stake|`Option<Stake>`|3|The stake of the operator in its own pool.|
|Delegated stakes|`IndexHeap<Stake, B_poolcap>`|4|The $B_{poolcap}$ largest stakes.|

#### Pool Key

A pool key is included in the next validator set index heap instead of a full pool to save space. Pool keys points to a pool and specifies an ordering of pools in the index heap.

|Field|Type|Suffix|
|---|---|---|
|Operator|`PublicAddress`|0|
|Power|`u64`|1|

### Stake

A stake is described in 2 fields:

|Field|Type|Suffix|Description|
|---|---|---|---|
|Owner|`PublicAddress`|0|The owner of the stake.|
|Power|`u64`|1|The amount of tokens in the stake.|

### Index Heap

`IndexHeap<E, S>` is a combination of a binary heap and a map that stores a maximum of `S` elements of type `E`.

This combination data structure implements all of the basic operations in *logarithmic* number of storage operations:
- Initialize / Destroy
- Insert-Extract
- Change-Power
- Get
- Remove
