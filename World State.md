# World State

|Revision no.|
|---|
|0|

We adopt Ethereum's terminology and call the singular user-visible state that the ParalelChain protocol maintains the "World State".

The world state is a set of key-value tuples representing the state of every *account*, stored inside a set of [Merkle Patricia Trie (MPT)](#merkle-patricia-trie) data structures.

This document specifies the structure and the contents of the world state, and is organized into five sections:
1. [Merkle Patricia Trie](#merkle-patricia-trie) specifies the variant of the MPT data structure that the ParallelChain protocol uses. The world state is comprised of two kinds of MPTs.
2. [Accounts Trie](#accounts-trie) specifies the first kind of MPT in the world state: the singular Accounts Trie.
3. [Storage Tries](#storage-tries) specifies the second kind of MPT: the Storage Trie. Unlike the accounts trie, which is singular, there can be many storage tries.
4. [Network Account Storage](#network-account-storage) specifies the contents of the storage trie under a special account called the Network Account.
5. [Genesis State](#genesis-state) specifies the initial contents of the entire world state, before any transaction is executed.

## Merkle Patricia Trie

A Merkle Patricia Trie is a data structure that maintains a mapping between keys and values and that has two useful cryptographic properties:
1. The entire content of an MPT is succinctly summarized in a single 32-byte `Keccak256` "root hash". Two MPTs are identical if they share the same root hash, and two MPTs are different if they have different root hashes. The root hash of the accounts trie after executing a block is included in its [block header](Blockchain.md#block-header).
2. A set of key-value pairs can be proven to be included in an MPT identified by a root hash without having send over the entire MPT. We plan to add RPCs in the future that benefit from this capability.

### Configuration

ParallelChain protocol MPTs are [paritytech/trie-db](https://docs.rs/trie-db/0.28.0/trie_db/index.html) v>=0.26 tries with the following configuration:

|Configuration|Value|
|---|---|
|Trie Layout|[`GenericNoExtensionLayout`](https://docs.rs/reference-trie/0.29.1/reference_trie/struct.GenericNoExtensionLayout.html)|
|Node Codec|[`ReferenceNodeCodecNoExt`](https://docs.rs/reference-trie/0.29.1/reference_trie/struct.ReferenceNodeCodecNoExt.html)|
|Hasher|[`KeccakHasher`](https://docs.rs/keccak-hasher/0.16.0/keccak_hasher/struct.KeccakHasher.html#)|

## Accounts Trie

The singular accounts trie is the "top-level" MPT in the world state, and instances of storage tries can be thought of as being "attached" to this trie via their root hashes ("storage hash"). The accounts trie stores a set of **accounts**.

### Account

An account is an abstract agent associated with some state, and which can trigger state changes in specific ways. 

The ParallelChain protocol has two main kinds of accounts, and they differ in how they trigger state changes: 
1. **External Accounts** trigger state changes by digitally signing transactions.
2. **Contract Accounts**  trigger state changes according to the logic in its code when they are [called](Runtime.md#call).

Every account is uniquely identified by a 32-byte sequence called a Public Address. The contents of a public address depends on whether the public address identifies an external account, or a contract account:
* An external account's address can be any Ed25519 public key.
* A contract account's address is the SHA256 hash of the concatenation of the `signer` and `nonce` fields[^1] of the transaction which [deployed](Runtime.md#deploy) it, as specified in the table below:

|Formula|Value|Description|
|---|---|---|
|$W_{contractaddr}(s, n)$|$SHA256((s, n))$|The address of a contract deployed in a transaction with signer = $s$ and nonce = $n$.|

[^1]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/2) to change $W_{contractaddr}$ to take in a command index to allow multiple deploy commands in a single transaction to succeed.

#### Special accounts

Besides the two main kinds of accounts, there are two special accounts:
1. The **Treasury Account** is a specific external account that receives a proportion of transaction base fees in the [charge phase of transaction execution](Runtime.md#charge), and is initialized with a large balance in the [genesis state](#genesis-state). The balance in the treasury account is used to fund ecosystem growth and development.
2. The **Network Account** is a pseudo-account that exists in the world state only so that its account storage (["network account storage"](#network-account-storage)) can be used to store DPoS-related state. It is not associated with an Ed25519 key.

The public addresses of the two special accounts are specified in the table below:

|Symbol|Address (Base 64)|Description|
|---|---|---|
|$W_{treasuryaddr}$|"j2tknpeg9VL24L9iAccipeuu5Q_B15ShRnBXjM-kAVM"|Address of the Treasury Account.|
|$W_{networkaddr}$|"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"|Address of the Network Account.|

### Account fields

Every account is associated with five fields in the accounts trie. The following table specifies these fields for a single account. Where the description table only describes the field's contents for specific kinds of accounts, the field is empty for all other kinds of accounts:

|Field number|Field|Type|Description|
|---|---|---|---|
|0|Nonce|`u64`|For an external accounts, the number of transations signed by the account so far in the blockchain history.|
|1|Balance|`u64`|The number of XPLL ("tokens") in the account, in grays.|
|2|Contract|`Vec<u8>`|For a contract account, a [valid contract](Contracts.md#contract-validity) deployed using the [deploy command](Runtime.md#deploy).|
|3|CBI version|`u16`|For a contract account, the major version of the CBI that the contract targets.|
|4|Storage hash|`CryptoHash`|For contract accounts and the network account, the root hash of their individual storage trie. This is initialized upon the first "set" on the associated account storage, and is updated upon every subsequent "set".|

#### Token balances

All token balances and amounts related to balances (e.g. stake powers) are stored in grays:

|Symbol|Formula|Description|
|---|---|---|
|$W_{xplltogray}$|$10^8$|Conversion factor between XPLL and gray. 1 XPLL = $W_{xplltogray}$ grays.|

### Accounts Trie MPT keys

Each account field is stored as a single key-value pair in the accounts trie MPT. For each pair, the key is formed by concatenating, in sequence:
1. The account's address.
2. The "visibility byte" `1u8`[^2].
3. The field number as `u8`.

Or equivalently, using the following formula:

|Symbol|Formula|Description|
|---|---|---|
|$W_{accfieldkey}(a, n)$|$a + 1u8 + n$|The MPT key in the accounts trie that stores the field with field number $n$ of the account with address $a$.|

[^2]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/4) to remove the visibility byte from MPT keys.

## Storage Tries

Each instance of storage trie stores an individual **account storage**.

### Account storage

An account storage is a collection of key-value pairs associated with a specific account that can hold "free-form" data, that is, arbitrary keys, and arbitrary values.

The main purpose of account storage is for contracts to use them to maintain state that persists between calls. Account storage is also used (as ["network account storage"](#network-account-storage)) to store DPoS-related state. The [contract binary interface](Contracts.md) provides host functions that allow contract code to "set", "get", or check the existence of key-value pairs in their contract account's storage.

### Storage Trie MPT keys

Each key-value pair provided through the "set", "get", and "contains" host functions of the contract binary interface is stored in as a single key-value pair in a storage trie MPT. For each pair, the key is formed by concatenating, in turn:

1. The "visibility byte"[^2] `0u8`.
2. The key provided in the "set", "get", or "contains" call.

Or equivalently, using the following formula:

|Symbol|Formula|Description|
|---|---|---|
|$W_{storagekey}(k)$|$0u8 + n$|The MPT key in the storage trie that stores a key $k$ provided through a CBI host function call or any other mechanism.|

Keys used to store fields and collections in network account storage are also formed using $W_{storagekey}$, but with $k$ taking the specific values specified in the next section.

## Network Account Storage

Besides state that is tied to particular external or contract accounts, the world state also keeps track of state that have significance to the entire network. This state is stored in the storage trie of a special account called the **network account**. The main purpose of this state is to implement [delegated proof of stake (DPoS)](Blockchain.md#delegated-proof-of-stake).

This section is organized into four subsections:
1. [Network Account Storage Fields](#network-account-storage-fields) specify the top-level fields stored in network account storage.
2. [Network Account Storage Pseudo-Types](#network-account-storage-pseudo-types) specify the "inner fields" of the "pseudo-types" stored in network account storage fields.
3. [Network Account Storage Storage Trie keys](#network-account-storage-storage-trie-keys) specify how the contents of network account storage are stored in storage key-value pairs.
4. [Index Heap Operations](#index-heap-operations) specify the sequence flows of operations on the "Index Heap" collection.

### Network Account Storage Fields

Network account storage consists of 6 numbered fields. Fields 0 to 4 contain pseudo-types called "collections", which are specified in the next section. The following table lists the 6 fields:

|Field number|Field|Pseudo-Type/Type|Description|
|---|---|---|---|
|0|Previous validator set|`IndexHeap<Pool,` $B_{vscap}$`>`|The validator set in the previous epoch.|
|1|Current validator set|`IndexHeap<Pool,` $B_{vscap}$`>`|The validator set in the current epoch.|
|2|Next validator set|`IndexHeap<PoolKey,` $B_{vscap}$`>`|The $B_{vscap}$ largest pools that are competing to become the next validator set.|
|3|Pools|`PublicAddress -> Pool`|All pools that are accepting stake and competing to become part of the next validator set, indexed by the operator of the pool.|
|4|Deposits|`PublicAddress -> Deposit`|All deposits.|
|5|Current epoch|`u64`|The current epoch.|

### Network Account Storage Pseudo-Types

A "pseudo-type" in the network account storage context refers to a collection of fields that are stored "within a field". Concretely, this means the fields in an instance of a pseudo-type is stored in a set of key-value pairs that each share a common prefix. 

How each field in an instance of a pseudo-type map to key-value pairs is specified in [network account storage storage trie keys](#network-account-storage-trie-keys). This section lists the fields of every pseudo-type.

#### Pool

Every pool is associated with five fields in network account storage:

|Field Number|Field|Pseudo-Type/Type|Description|
|---|---|---|---|
|0|Operator|`PublicAddress`|The operator of the pool.|
|1|Power|`u64`|The sum of the powers' of the pool's operator stake and delegated stakes.|
|2|Commission rate|`u8`|The percentage (0-100%) of the epoch’s issuance rewarded to the pool that will go towards the operator’s stake (or their balance, if the operator did not stake to itself).|
|3|Operator stake|`Option<Stake>`|The stake of the operator in its own pool.|
|4|Delegated stakes|`IndexHeap<Stake,`$B_{poolcap}$`>`|The $B_{poolcap}$ largest delegated stakes in this pool.|

#### Pool Key

The next validator set field in network account storage stores Pool Keys instead of Pools in order to save space. Each Pool Key in the next validator set field corresponds to a pool that already exists in the Pools field, so the other fields of the Pool represented by a Pool Key can be retrieved by indexing into the Pools field using the Pool Key's operator field.

Pool keys have two fields:

|Field Number|Field|Type|Description|
|---|---|---|---|
|0|Operator|`PublicAddress`|The operator of the pool.|
|1|Power|`u64`|The sum of the powers' of the pool's operator stake and delegated stakes.|

#### Stake

Every stake is associated with two fields in network account storage:

|Field Number|Field|Type|Description|
|---|---|---|---|
|0|Owner|`PublicAddress`|The owner of the stake.|
|1|Power|`u64`|1|The number of tokens in the stake.|

#### Deposit

Every deposit is associated with two fields in network account storage:

|Field Number|Field|Type|Description|
|---|---|---|---|
|0|Balance|`u64`|The number of tokens in the deposit.|
|1|Automatically Stake Rewards?|`bool`|Whether rewards received by the stake associated with this deposit at the end of one epoch should be added to the deposit's stake in the next epoch.| 

#### Mapping (`T -> U`)

Mappings are pseudo-types with as many fields as there are unique instances of the key type `T`, each field storing the type/collection `U`. For example, the pseudo-type `PublicAddress -> Deposit` has $2^{256}$ fields, since there are $2^{256}$ possible public addresses.

#### Index Heap (`IndexHeap<Item, Capacity>`)

Index Heaps are min-heaps with a specified "item" and "capacity". The heap invariant that Index Heaps maintain on its items allow [CRUD operations](#index-heap-operations) to be done on them more efficiently, reducing the gas used to execute DPoS-related commands. Possible items and capacity are:
* Item: Pool, Pool Key, or Stake.
* Capacity: $B_{vscap}$ or $B_{poolcap}$.

Items are pseudo-types with a "key address" property. An Index Heap orders its item in ascending (lowest-is-root) order of key addresses. The key address for each Item pseudo-type are:
* Pool: Operator.
* Pool Key: Operator.
* Stake: Owner.

Index heaps have three fields:

|Field Number|Field|Pseudo-Type/Type|Description|
|---|---|---|---|
|0|Length|`u32`|Number of items currently in the index heap.|
|1|Key Address to Index|`PublicAddress -> u32`|A mapping between a key address of an index heap item and its index in the index heap.|
|2|Index to Item|`u32 -> Item`|A mapping between an index and an item in the index heap.|

### Network Account Storage storage trie keys

Each "type"-field in network account storage (i.e., a field that contains an instance of a type, instead of a pseudo-type) corresponds to a single key-value pair in the network account's storage trie. For each pair, the key is determined by the sequence of "pseudo-type"-fields that contain the type-field.

For example, the storage trie key of the power field of a pool key with key address $k$ in the next validator set is: `2u8` + `2u8`, $k$ + `1u8`, where the terms in the addition are, in order:
1. The field number of the next validator set field in network account storage.
2. The field number of the index to item field in the index-heap pseudo-type.
3. The field number of the key address in the mapping pseudo-type.
4. The field number of the power field in the pool key pseudo-type.

### Index Heap Operations

This section defines the sequence flows of operations on Index Heaps for use in the execution of [DPoS-related commands](Blockchain.md#staking-commands).

Even though these sequences flows are not necessarily the most efficient way to implement the business logic, it is important to follow them because in particular, different sequences of world state operations will cost different amounts of gas.

The sequence flows in this section are written with the following conventions:
- Network account storage is modelled as a type, and its operations are modelled as methods taking a `&mut self` receiver.
- Collections are also likewise modelled as types.
- "Get" accesses on a field are denoted with function call syntax and returns an `Option<T>` (e.g., `let _: Option<u32> = self.length()`) when the field could be empty at the point of access, and with indexing syntax (e.g., `let _: Self::Item = self.index_to_item[index]`) when the field will never be empty at the point of access.
- "Set" accesses on a field are always denoted using indexing syntax. (e.g., `self.index_to_item[index] = item`).
- "Get" accesses on a collection gets all of the collection's inner fields except inner Index Heaps (e.g., `let _: Pool = self.index_to_item[index]` gets $32 + 8 + 1 + (1 + 32 + 8) = 82$ bytes, the terms in the addition corresponding to the pool fields "operator", "power", "commission rate", and "own stake" respectively).
- Likewise, "set" accesses on a collection sets all of the collection's inner fields except inner Index Heaps.

##### Insert-and-Extract

Insert an item into the Index Heap, removing the root (lowest power) item if necessary to keep the Index Heap's length at or below capacity.

```rust 
fn insert_and_extract(&mut self, item: Self::Item) -> Result<Option<Item>, PowerTooLow> {
    let length = self.length();
    if length == 0 {
        self.insert(item).unwrap();
        Ok(None)
    }
    
    else {
        let root_item = self.index_to_item[0];
    
        if length == self.capacity {
            if item.power < root_item.power {
                Err(PowerTooLow)
            } else {
                self.extract();
                self.insert(item);
                Ok(Some(root_item))
            }
        }

        else {
            self.insert(item);
            Ok(None)
        } 
    }
}
```

##### Insert

Insert an item into the Index Heap, returning an error if the heap's length is already at capacity.

```rust
fn insert(&mut self, item: Self::Item) -> Result<(), HeapFull> {
    let length = self.length.unwrap_or(0);
    if length == Self::Capacity {
        return Err(HeapFull);
    }
    
    // Insert the item at the end of the index heap.
    self.index_to_item[length] = item;
    self.key_address_to_index[item.index_heap_key_address()] = item.key_address();
    self.length = length + 1;
    
    // Up-heapify the item.
    self.up_heapify(length);
    
    Ok(());
}
```

##### Up-Heapify

Compare the item at the target index in the heap with its parent and swap them if an item is has a lower power than its parent. Continues iteratively until the root item.

```rust
fn up_heapify(&mut self, mut target_index: u32) {
    // The item at index 0 is at the root of the heap. It cannot
    // be up-heapified any further.
    if index == 0 {
        return
    }
   
    let parent_index = (target_index - 1) / 2;
    let target_item = self.index_to_item[target_index];
    let parent_item = self.index_to_item[parent_index];

    // If the target item is smaller than its parent, swap their
    // positions.
    if target_item.power < parent_item.power {
        self.index_to_item[target_index] = parent_item;
        self.key_address_to_index[parent_item.index_heap_key_address()] = target_index;
        self.index_to_item[parent_index] = target_item;
        self.key_address_to_index[target_item.index_heap_key_address()] = parent_index;
    } 

    // Else, the target item is already fully up-heapified.
    else {
        break
    }
}
```

##### Extract

Delete and return the root item of the heap, if there are any items in the heap.

```rust
fn extract(&mut self) -> Option<Self::Item> {
    let length = self.length.unwrap_or(0);
    
    // If the length of the index heap is 0, there are no elements to
    // extract.
    if length == 0 {
        return None;
    }
    
    let root_item = self.index_to_item[0];
    
    // If the index heap only contains one item, delete it and return
    // it.
    if length == 1 {
        self.length = 0;
        self.index_to_item[0].delete();
        self.key_address_to_index[root_item.index_heap_key_address()].delete();
    }
    
    // The root-item item is get a second time. This is not
    // logically necessary, but implementations must charge for this
    // operation twice in this function in order to get the same "gas 
    // used" result as the reference implementation.
    let root_item = self.index_to_item[0];
    
    // Delete the root item and move the last item to the root
    // of the heap.
    let last_item = self.index_to_item[length - 1];
    self.key_address_to_index[root_item.index_heap_key_address()].delete();
    self.index_to_item[length - 1].delete();
    self.index_to_item[0] = last_item;
    self.key_address_to_index[last_item.index_heap_key_address()] = 0;
    self.length = length - 1;
    
    // Down-heapify the root.
    self.down_heapify(0, length - 1);
}
```

##### Down-Heapify

Compare the item at the target index in the heap with its children and swap the item with the largest-powered of its children if an item has a larger power than at least one of its children. Continues iteratively until reaching an item without children.

```rust
fn down_heapify(&mut self, mut target_index: u32, length: u32) {
    loop {
        // In each iteration of this loop, we consider three items:
        // 1. The item identified by the target index (the "target item").
        // 2. The target item's left child.
        // 3. The target item's right child.
        let left_child_index = 2 * target_index + 1;
        let right_child_index = 2 * target_index + 2;
        
        // We initially assume that the target item is the smallest
        // of the three items in terms of power.
        let cur_smallest_item_index = target_index;
        
        // Check whether the left child item has a smaller power than
        // the current smallest item.
        if left_child_index < length {
            let cur_smallest_item = self.index_to_item[cur_smallest_item_index];
            let left_child_item = self.index_to_item[left_child_index];
            if left_child_item.power() < cur_smallest_item.power() {
                cur_smallest_item_index = left_child_index;
            }
        }
        
        // Check whether the right child item has a smaller power than
        // the current smallest item.
        if right_child_index < length {
            let cur_smallest_item = self.index_to_item[cur_smallest_item_index];
            let right_child_item = self.index_to_item[left_child_index];
            if right_child_item.power() < cur_smallest_item.power() {
                cur_smallest_item_index = right_child_index;
            }
        }
        
        // If the target item is not the smallest of the three items,
        // swap the target item with the smallest of the three items.
        if target_index != cur_smallest_item_index {
            let target_item = self.index_to_item[target_index];
            let smallest_item = self.index_to_item[cur_smallest_item_index];
            
            self.index_to_item[target_index] = smallest_item;
            self.key_address_to_index[smallest_item.index_heap_key_address()] = target_index;
            
            self.index_to_item[cur_smallest_item_index] = target_item;
            self.key_address_to_index[target_item.index_heap_key_address()] = cur_smallest_item_index;
            
            // Continue the loop, further down-heapifying the target
            // item.
            target_index = cur_smallest_item_index;
        }
        
        // Else, the heap property is already satisfied and down-heapify
        // is complete.
        else {
            break;
        }
    }
}
```

##### Update

Update an item if it is currently in the trie (where "currently in the trie" means that an item with the same index heap key address is currently in the trie).

```rust
fn update(&mut self, item: Self::Item) {
    let length = self.length.unwrap_or(0);
    if let index = Some(index) = self.key_address_to_index(item.index_heap_key_address()) {
        let cur_item = self.index_to_item[index];
        
        if item.power > cur_item.power {
            self.index_to_item[index] = item;
            self.down_heapify(index, length);
        } else if item.power < cur_item {
            self.index_to_item[index] = item;
            self.up_heapify(index, length);
        }
    } 
}
```

##### Delete

Delete the item identified by the given key address if it is currently in the trie.

```rust
pub fn delete(&mut self, key_address: PublicAddress) {
    let length = self.length();
    
    if let index = Some(index) = self.key_address_to_index(key_address) {
          
         // If the item to be deleted is at the root of the heap, just
         // extract it.
         if index == 0 {
             self.extract();
         }

         // Else, if the item to be deleted is the last item in the heap, 
         // just delete it and its key address-to-index mapping.
         else if index == length - 1 {
            self.key_address_to_index[key_address].delete()
            self.index_to_item[key_address].delete()
            self.length = length - 1;
             
        // Otherwise, delete the item and put the last item in the heap in its
        // place.
        } else {
            let item = self.index_to_item[index];      
            let last_item = self.index_to_item[length - 1];
            self.key_address_to_index[item.index_heap_key_address()].delete();
            self.index_to_item[length - 1].delete();
            self.index_to_item[0] = last_item;
            self.key_address_to_index[last_item.index_heap_key_address()] = 0;
            self.length = length - 1;
            
            // If the last item's power is larger than the deleted item, down-
            // heapify it.
            if last_item.power > item {
                self.down_heapify(index, length - 1);
            } 
            
            // Else, if the last item's power is smaller than the deleted item,
            // up-heapify it.
            else if last_item.power < item {
                self.up_heapify(index, length - 1);
            }
        }
        
    }
}
```

## Genesis State

Before the genesis block (the block with block height 0) is executed, the World State is empty except for the data specified in this section.

### Genesis Balances

Three accounts are assigned an initial balance:

|Account Address|Balance|
|---|---|
|$W_{treasuryaddr}$|3047500000000000|
|"okncC7Xu5z0soYnDr-BsKnzPLnXLVDycvKAupN47cIc"|375000000000000|
|"bK85cvUnpY4gWs27R1aIHNBYAUjvzshFGknMXHK72Qs"|8977500000000000|

### Genesis Epoch

The "current epoch" field in network account storage is initialized to `0`.

### Genesis Validators

Network account storage is initialized to have 10 validators by setting the following network account storage fields:
* Current Validator Set.
* Previous Validator Set.
* Pools.
* Deposits.

The operator addresses of the genesis validators, as well as their initial deposit balance, pool own stake, pool commission rate, and deposit "automatically stake rewards?" setting are specified in the table below:

|Operator|Deposit Balance/Power|Commission Rate|Automatically stake rewards?|
|---|---|---|---|
|"IiJoIUHLuuBUHyGHU2_rGSyoHBZ3LQNqI5dgo6ff0k4"|10000000000000|0|False|
|"9NK8lCsbuJtcnLxnXzmPH12RnfOOhDCLxvf5hqa6UWk"|10000000000000|0|False|
|"N6G0pq-uFEVt-OvDJZJbRoxs1wf7bsQ25BL3ImedBVs"|10000000000000|0|False|
|"\_G2mANINJZUIdukS6B3zZ43st9HDe3HjShgJOUNMiAc"|10000000000000|0|False|
|"ILDrzBVvVRqhLNGsE7eJJW7KA4vLU20fKWL8oZbS8Gs"|10000000000000|0|False|
|"R5aexUGcjnUwYuy7JrffNb8hsdmEllWEoMoodJ4s3xQ"|10000000000000|0|False|
|"o7iyYZaaVT6Z9U5HMjR01Zj1_Q5ejzrsrVQhA03BmII"|10000000000000|0|False|
|"zFPda0ZA056Coiq5CG9L1iRbbmufcpIAn9rzjzHqgAA"|10000000000000|0|False|
|"x7eiywH_8YVHQkSgjZk3EXdLU3FGo4VaV_6qi-hzOKI"|10000000000000|0|False|
|"oK8Kvd-2cWYloQaPNlGtG3Q5dV6JFKzVrXOAhBRt5hs"|10000000000000|0|False|
