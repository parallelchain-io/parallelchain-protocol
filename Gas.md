# Gas (V1)

The cost of executing a transaction is measured in gas, and counted by updating the $gc$ state variable in the runtime.

We charge gas for 5 categories of operations:
- [WASM opcode execution](#wasm-opcode-execution) inside contract calls.
- [Reading and writing WASM memory from host functions](#accessing-wasm-memory-from-host-functions) inside contract calls.
- [Transaction-related data storage](#transaction-related-data-storage).
- [World state storage and access](#world-state-storage-and-access).
- [Cryptographic operations](#cryptographic-operations) inside contract calls.

These categories are not an exhaustive enumeration of all of the tasks that a validator does to maintain service for users, but are what we deem to be the tasks that are either most expensive, or most variable, and therefore most vulnerable to abuse by users if they are free.  

The following sections specify the gas costs of each category of operations, and then defines the [transaction inclusion cost](#transaction-inclusion-cost) $G_{txincl}$, the minimum gas limit that a transaction must have to be included in the blockchain. 

## WASM opcode execution

The basic principle in deciding the cost of executing a WASM opcode is *1 CPU cycle, 1 gas unit*. 

Precisely, the cost of each opcode was decided by compiling each opcode one-by-one into x86 assembly using the online [WebAssembly Explorer](https://mbebenita.github.io/WasmExplorer/) tool, and then counting the latency of the generated x86 assembly instructions in a contemporary Intel Core processor (Coffee Lake) using [Fog's instruction tables](https://www.agner.org/optimize/instruction_tables.pdf) for reference.

For simplicity, we ignore the memory pressure caused by executing certain opcodes and highly CPU-specific details such as instruction pipelining and energy use, and we also do not specify the gas costs of [illegal opcodes](CBI#appendix-illegal-opcodes).

The gas cost of executing every legal opcode is specified at the end of this document in the [Appendix](#appendix-wasm-opcode-gas-schedule) for readability.

## Accessing WASM memory from host functions

We assume that the cost of loading a byte from a guest WASM module's memory into host code is roughly 1/8th as the cost of loading 8 bytes from the module's memory into a virtual register using the I64Load instruction, and the converse for storing a byte. 

|Name|Formula|Description|
|---|---|---|
|$G_{wread}(len)$|$max(\lceil len/8 \rceil \times Cost_{I64Load}, 1)$|Cost of reading $len$ bytes from the guest's memory.|
|$G_{wwrite}(len)$|$max(\lceil len/8 \rceil \times Cost_{I64Store}, 1)$|Cost of writing $len$ bytes into the guest's memory.|

## Data storage

As alluded to previously, we charge gas to store two kinds of data: 
1. Transaction-related data (Transactions and their Receipts) which are stored in Blocks.
2. Tuples in the world state.

In deciding the costs of both kinds of data storage, we use [Ethereum](https://ethereum.github.io/yellowpaper/paper.pdf) as the baseline and then make adjustments based on a specific desideratum: we want applications to use less storage. 

To achieve this desideratum, we set the cost of writing 1 byte as a multiple of the cost of 1 CPU cycle to be 2 or 4x greater than Ethereum for transaction-related data and world state data respectively.

## Transaction-related data storage

Transaction-related data includes every byte in the Borsh-serialization of transactions (and its fields), and receipts (and its fields). Each byte of transaction-related data costs $G_{txdata}$ to store.

|Formula|Value|Description|Ethereum counterpart|
|---|---|---|---|
|$G_{txdata}$|30|Cost of including 1 byte of data in a Block as part of a transaction or a receipt.|$G_{txdatanonzero} = 16$|

The cost of storing a transaction and the fixed-sized component of its receipt is known before the transaction is charged in the transaction's [inclusion cost](#transaction-inclusion-cost). On the other hand, the size of the dynamic-sized contents of command receipts, namely return values and logs, are dynamically charged during execution.

## Transaction inclusion cost

An amount of gas called the "transaction inclusion cost" is charged before every other step in a transaction's execution by initializing the [$gc$ Runtime state variable](Runtime.md#state-variables) to it:

|Formula|Value|Description|
|---|---|---|
|$G_{txincl}(txn)$|$[serialize(txn).len() + G_{minrcpsize}(txn.commands)] \times G_{txdata} + 5 \times [G_{sget}(G_{acckeylen}, 8) + G_{sset}(G_{acckeylen}, 8, 8)]$|The transaction inclusion cost of a transaction $txn$.| 

The above formula consists of two terms that are added together:
1. The first term accounts for storing the transaction and its "minimal-size" receipt.
2. The second term accounts for getting and then setting 5 pair of keys in the Accounts Trie in the pre-charge and charge execution phases:
    1. The signer's nonce.
    2. The signer's balance, in the pre-charge phase.
    3. The signer's balance again, in the charge phase.
    4. The proposer's balance.
    5. The treasury's balance.

The minimal-size receipt of any transaction with $n$ commands is a `Vec<CommandReceipt>` containing $n$ identical command receipts, each with an empty return value and log:

|Formula|Value|Description|
|---|---|---|
|$G_{minrcpsize}(n)$|$4 + n \times G_{cmdrcpminsize}$|Size of a receipt containing $n$ minimal-sized command receipt.|
|$G_{mincmdrcpsize}$|$17$|Size of a single minimal-sized command receipt.|

## World state storage and access

An MPT is a dictionary with three fundamental access operations:
1. **set**: create a mapping between a variable length and a variable length value. Setting a key to an empty value is equivalent to deleting a key.
2. **get**: get the value mapped to a key.
3. **contains**: check whether a key is mapped to a value.

This section defines cost formulas for each fundamental access operation first on the Accounts Trie, and then a Storage Trie, and finally on a generic MPT.

### Accounts Trie operations

Getting, setting, and checking for the existence of a key-value pair in the Accounts Trie simply entails doing the corresponding operation in the dedicated Accounts Trie MPT.

#### Get

|Symbol|Formula|Description|
|---|---|---|
|$G_{at,get}(k, a)$|$G_{mpt,get}(k, a)$|Cost of getting a key of length $k$ that is currently mapped to a value of length $a$ from the Accounts Trie.|

Getting contract code from the Accounts Trie is discounted:

|Symbol|Formula|Description|
|---|---|---|
|$G_{at, getcontractdisc}$|50%|Proportion of $G_{at, get}$ which is discounted if the tuple contains a contract.|None|

#### Set

|Symbol|Formula|Description|
|---|---|---|
|$G_{at,set}(k, a, b)$|$G_{mpt,set}(k, a, b)$|Cost of setting a key of length $k$ to a value of length $b$ in the Accounts Trie given that the key is currently set to a value of length $a$.|

#### Contains

|Symbol|Formula|Description|
|---|---|---|
|$G_{at,contains}(k)$|$G_{at,get}(k, 0)$|Cost of checking whether a key with length $l$ is set to a value in the Accounts Trie.|

### Storage Trie operations

The cost of getting, setting, or checking for the existence of a key-value pair in a storage trie where key is of length $k$ is equivalent to the cost of doing the same operation on an imaginary "combined MPT" with a key of length $G_{at, klen} + k + 32$. Where $G_{at, klen}$:

|Formula|Value|Description|
|---|---|---|
|$G_{at, keylen}$|33|The length of Accounts Trie keys.|

The first two terms of the key in the combined MPT simulates an architecture that has each storage MPT 'attached' as a subtrie of the Accounts Trie on the tuple that stores the account's storage hash, while the $+ 32$ term is there for legacy reasons[^1].

[^1]: We [plan](https://github.com/parallelchain-io/parallelchain-protocol/issues/3) to remove the $+ 32$ term.

#### Get

|Symbol|Formula|Description|
|---|---|---|
|$G_{st,get}(k, a)$|$G_{mpt,get}(G_{at, keylen} + k + 32, a)$|Cost of getting a key of length $k$ that is currently mapped to a value of length $a$ from a Storage Trie.|

#### Set

|Symbol|Formula|Description|
|---|---|---|
|$G_{st,set}(k, a, b)$|$G_{mpt,set}(G_{at, keylen} + k + 32, a, b)$|Cost of setting a key of length $k$ to a value of length $b$ in a Storage Trie given that the key is currently set to a value of length $a$.|

##### Contains

|Symbol|Formula|Description|
|---|---|---|
|$G_{st,contains}(k)$|$G_{st,get}(k, 0)$|Cost of checking whether a key with length $l$ is set to a value a Storage Trie.|

### MPT operations

To decide the cost of each MPT operation, we imagine how a simple implementation of an MPT may implement the operation.

#### Get

To get a key of length $k$ which is mapped to a value of length $a$:
1. Traverse down the MPT until we reach a matching node or dead end.
2. Read and return its value, or none, if we reached a dead end.

|Symbol|Formula|Description|
|---|---|---|
|$G_{mpt,get,v2}(k, a)$|$G_{mpt,get1}(k) + G_{mpt,get2}(a)$|Cost of getting a key of length $k$ that is currently is mapped to a value of length $a$ from an MPT.|
|$G_{mpt,get1}(k)$|$k \times G_{mpt, traverse}$|Cost of step 1 of get.|
|$G_{mpt,get2}(a)$|$a \times G_{mpt, read}$|Cost of step 2 of get.|

#### Set

To set a key of length $k$, currently set to a value of length $a$, to a new value of a set $b$:
1. Get the key.
2. Delete the old value for a refund.
3. Write a new value.
4. Recompute node hashes until the root.

|Symbol|Formula|Description|
|---|---|---|
|$G_{mpt,set}(k, a, b)$|$G_{mpt, set1}(k, a) + G_{mpt, set2}(k, a, b) + G_{mpt, set3}(b) + G_{mpt, set4}(k)$|Cost of setting a key of length $k$ to a value of length $b$ in an MPT given that the key is currently set to a value of length $a$.|
|$G_{mpt,set1}(k, a)$|$G_{mpt,get}(k, a)$|Cost of step 1 of set.|
|$G_{mpt,set2}(k, a, b)$|Given in the paragraph below this table.|Cost of step 2 of set.|
|$G_{mpt,set3}(b)$|$b \times G_{mpt, write}$|Cost of step 3 of set.|
|$G_{mpt,set4}(k)$|$k \times G_{mpt, rehash}$|Cost of step 4 of set.|

$$
G_{mpt, set2}(k, a, b) = -1 \times \begin{cases} 
a \times G_{mpt, write} \times G_{mpt, refund} & \text{if $a \ge 0$, $b > 0$}, \\
(k + a) \times G_{mpt, write} \times G_{mpt, refund} & \text{if $a > 0$, $b = 0$}, \\
0 & \text{if $a = 0$, $b = 0$}, \\
\end{cases} 
$$

#### Constants

|Formula|Value|Description|Ethereum counterpart|
|---|---|---|---|
|$G_{mpt, write}$|2500|Cost of writing a single byte into an the backing storage of an MPT.|$G_{sset} = 20000$ for storing 32 bytes, so $625$ per byte.|
|$G_{mpt, read}$|50|Cost of reading a single byte from the backing storage of an MPT.|$G_{coldsload} = 2100$ for loading 32 bytes, so roughly $65$ per byte.| 
|$G_{mpt, traverse}$|20|Cost of traversing 1 byte (2 nibbles) down an MPT.|None|
|$G_{mpt, rehash}$|130|Cost of traversing 1 byte up (2 nibbles) and recomputing the SHA256 hashes of 2 nodes in an MPT after it or one of its descendants is changed.|None|
|$G_{mpt, refund}$|50%|Proportion of the cost of writing a tuple into an MPT that is refunded when that tuple is re-set or deleted.|$R_{sclear} = 15000$, which ends up being [50%](https://ethereum.stackexchange.com/a/40972).|

## Cryptographic operations  

|Formula|Value|Description|
|---|---|---|
$G_{wsha256}(len)$|$len \times 16$|Cost of computing the SHA256 hash over a message of length $len$|
$G_{wkeccak256}(len)$|$len \times 16$|Cost of computing the Keccak256 hash over a message of length $len$|
$G_{wripemd160}(len)$|$len \times 16$|Cost of RIPEMD160 hash over a message of length $len$|
$G_{wvrfy25519}(len)$|$1400000 + len \times 16$|Cost of verifying whether an Ed25519 signature over a message of length $len$ was produced by a given address|

These per-byte costs were decided using semi-standardized experiments:

### Deciding on the cost of hashes

The methodology we used to decide $C_{wsha256} = C_{wkeccak256} = C_{wripemd160} = len \times 16$ was to:
1. Select a popular Rust implementation of SHA256 (and Keccak256, and RIPEMD160).
2. Write a simple Rust binary that uses the implementation to SHA256-hash a message.
3. Compile the Rust binary on Ubuntu 20.04 LTS running on a modern platform (Intel Core Coffee Lake) with `--release` optimizations enabled.
4. Inspect the generated assembly to count how many CPU cycles it takes to hash the message (per byte), assuming no pipelining.

### Deciding on the cost of Ed25519 signature verification

Implementation details of Ed25519 verification made the methodology we used to decide on the cost of hashes difficult to apply for deciding $C_{wvrfy25519}$, so we used a different experiment.

The idea for this experiment is to decide on a rough ratio estimate of how much more efficient running a cryptographic operation is in native code than running it inside WASM. Once we have this ratio, we gas meter Ed25519 signature verification in WASM, then apply the ratio to estimate what it would have costed to verify a signature in native code. The experiment has two parts.

The first part:

1. Compile the Rust binary we wrote previously for the SHA256 experiment into wasm32-unknown-unknown target with `--release` optimizations.
2. Execute the WASM module, gas metering opcodes according to our schedule. 

We found that the execution consumed 80 gas, which we treated as equivalent to 80 CPU cycles. This led us to the rough estimation that running a cryptographic operation in native code is 5x faster (80/16) than running it in a contract. 

The second part:

3. Select a popular Rust implementation of Ed25519 verification.
4. Write a trivial Rust binary that uses the implementation to verify signatures of varying lengths.
5. Compile the Rust binary to a wasm32-unknown-unknown target with `--release` optimizations.
6. Execute the WASM module with varying message lengths, gas metering opcodes according to our schedule.

We found that the base cost of verifying an Ed25519 signature in a contract is 7,000,000. Applying the 5x faster estimation from before, this led us to setting a base cost of 1,400,000 in $G_{wvrfy25519}$. The $len * 16$ formula comes from the fact that Ed25519 verification involves a SHA512 as a preparation step. 

## Appendix: WASM opcode gas schedule

### Constants

| Opcodes | Gas Cost |
|:--- |:--- |
I32Const  |0|
I64Const  |0|

### Type parameteric operators

| Opcodes | Gas Cost | 
|:--- |:--- |
Drop |2| 
Select |3|

### Flow control

| Opcodes | Gas Cost |
|:--- |:--- |
Nop, Unreachable, Else, Loop, If  |0| 
Br, BrTable, Call, CallIndirect, Return |2|
BrIf |3|
  
### Registers

| Opcodes | Gas Cost |
|:--- |:--- |
GlobalGet, GlobalSet, LocalGet, LocalSet  |3|
  
### Reference Types

| Opcodes | Gas Cost |
|:--- |:--- |
RefIsNull, RefFunc, RefNull, ReturnCall, ReturnCallIndirect |2| 
  
### Exception Handling

| Opcodes | Gas Cost |
|:--- |:--- |
CatchAll, Throw, Rethrow, Delegate |2|

### Bulk Memory Operations

| Opcodes | Gas Cost |
|:--- |:--- |
ElemDrop, DataDrop  |1|
TableInit |2| 
MemoryCopy, MemoryFill, TableCopy, TableFill  |3| 

### Memory Operations

| Opcodes | Gas Cost |
|:--- |:--- |
|I32Load, I64Load, I32Store, I64Store, I32Store8, I32Store16, I32Load8S, I32Load8U, I32Load16S, I32Load16U, I64Load8S, I64Load8U, I64Load16S, I64Load16U, I64Load32S, I64Load32U, I64Store8, I64Store16, I64Store32 |3|   

### 32 and 64-bit Integer Arithmetic Operations

| Opcodes | Gas Cost |
|:--- |:--- |
|I32Add, I32Sub, I64Add, I64Sub, I64LtS, I64LtU, I64GtS, I64GtU, I64LeS, I64LeU, I64GeS, I64GeU, I32Eqz, I32Eq, I32Ne, I32LtS, I32LtU, I32GtS, I32GtU, I32LeS, I32LeU, I32GeS, I32GeU, I64Eqz, I64Eq, I64Ne, I32And, I32Or, I32Xor, I64And, I64Or, I64Xor, |1|
|I32Shl, I32ShrU, I32ShrS, I32Rotl, I32Rotr, I64Shl, I64ShrU, I64ShrS, I64Rotl, I64Rotr,  |2|
|I32Mul, I64Mul |3|
|I32DivS, I32DivU, I32RemS, I32RemU, I64DivS, I64DivU, I64RemS, I64RemU |80|
|I32Clz, I64Clz |105|
  
### Type Casting & Truncation Operations

| Opcodes | Gas Cost |
|:--- |:--- |
|I32WrapI64, I32Extend8S, I32Extend16S, I64ExtendI32S, I64ExtendI32U, I64Extend8S, I64Extend16S, I64Extend32S |3| 
