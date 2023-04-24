# Contracts

|Revision no.|CBI version|
|---|---|
|0|0.0|

Contracts are code that are deployed into contract accounts. They have exclusive write access into their account's state.

This document specifies the latest version of the Contract Binary Interface (CBI): version 0.0.

The CBI is a versioned standard that specifies three things:
1. [Contract validity](#contract-validity): properties that code must satisfy to be deployed as a contract and be called.
2. [Host functions](#host-functions): a set of functions that the host process must make available for the guest contract to call.
3. [Guest functions](#guest-functions): a set of functions that the guest contract must make available for the host process to call.

We begin by quickly defining contract validity. Then, we specify general properties that host function executions must satisfy. Finally, we list all available host functions and guest functions, describing their executions in more detail where there is room for ambiguity.

## Contract validity

To be deployed and called as a contract, code must:
- Be "valid" WebAssembly modules, as described in the [WebAssembly 2.0 specification](https://webassembly.github.io/spec/core/intro/overview.html#semantic-phases).
- Not import any function besides those specified in [host functions](#host-functions).
- Not contain any [illegal opcodes](#appendix-illegal-opcodes). These opcodes may cause nondeterminism. 
- Export the [call](#call) guest function (contracts that do note export call can be deployed, but cannot be called).

## General properties of host function executions

### View calls

The RPC offers a [view](RPC.md#view) procedure, which calls a contract outside the context of a transaction. View calls cannot change the world state or get any transaction or block-related contexts. 

To enforce this, host functions that are marked as not available in view calls in the list must panic if called in a view call.

### Gas metering

In contract calls and inside host functions, we charge for four kinds of operations:
1. [Reading from and writing to guest memory from host code](gas#reading-wasm-memory-in-host-functions-inside-a-contract-call).
2. [Storage gets and sets](gas#world-state-access).
3. [Returning values and pushing logs to the receipt](gas#transaction-related-data-storage).
4. Computation of [cryptographic operations](gas#cryptographic-operations-inside-work-steps).

We *do not* charge gas for reading host function arguments or for writing host function return values.

### Passing big values between host and guest

Because WebAssembly supports a limited number of types, we have to pass values larger than 8 bytes ("big values") between host and guest by reading and writing into the guest function's memory.

As hard rules:
1. A host function that *takes* a big value will have a `*_ptr` argument and a matching `*_len` argument. The host function reads this big value by reading `*_len` bytes from the guest's linear memory starting from the position specified by `*_ptr`.
2. A host function that *returns* a big value will have a `*_ptr_ptr` argument. To return the big value, the host function will call the guest's `alloc` function and write the big value in the allocation, and then write the start of the allocation in 4 bytes starting from the position specified by `*_ptr_ptr`.
3. If the big value that the host function returns has a variable length, then the function will return the length.

## List of host functions

### World state accessors 

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Set|`fn set(key_ptr: *const [u8], key_len: u32, val_ptr: *const [u8], val_len: u32);`|Set a key-value pair in the current account's storage.|No|
|Get|`fn get(key_ptr: *const [u8], key_len: u32, val_ptr_ptr: *mut *const [u8]) -> i64;`|Get a key in the current account's storage. If the key is not set, then this function returns -1, otherwise it returns the length of the value.|Yes|
|Balance|`fn balance() -> u64;`|Get the current account's balance|Yes|
|Get from network storage|`fn get_network_storage(key_ptr: *const u8, key_len: u32, value_ptr_ptr: *const u32) -> i64;`|Get a key in the network account's storage. If the key is not set, then this function returns -1, otherwise it returns the length of the value.|Yes|

### Block context getters 

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Block height|`fn block_height() -> u64;`|Get the height of the block that the transaction which triggered the current call is part of.|No|
|Block timestamp|`fn block_timestamp() -> u32;`|Get the timestamp of the block that the transaction which triggered the current call is part of.|No|
|Previous block hash|`fn prev_block_hash(hash_ptr_ptr: *mut *const CryptoHash);`|Get the block hash of the parent of the block that the transaction which triggered the current call is part of.|No|

### Call context getters

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Calling account|`fn calling_account(address_ptr_ptr: *mut *const PublicAddress);`|Get the address of the calling account, which is either an external account (if this contract was entered directly from a Call command), or a contract account (if it is an internal call).|No|
|Current account|`fn current_account(address_ptr_ptr: *mut *const PublicAddress);`|Get the address of the current contract.|Yes|
|Method|`fn method(method_ptr_ptr: *mut *const [u8]) -> u32;`|Get the method field of the CallInput of the current call.|Yes|
|Arguments|`fn arguments(arguments_ptr_ptr: *mut *const [u8]) -> u32;`|Get the arguments field of the CallInput of the current call.|Yes|
|Amount|`fn amount() -> u64;`|Get the amount field of the CallInput of the current call.|No| 
|Is internal call?|`fn is_internal_call() -> u32;`|Returns whether the current call is an internal call.|Yes|
|Transaction hash|`fn transaction_hash(hash_ptr_ptr: *mut *const CryptoHash);`|Get the hash of the transaction which triggered this call.|No|

### Internal command triggers

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Call|`fn call(call_input_ptr: *const [u8], call_input_len: u32, rval_ptr_ptr: *mut *const [u8]) -> u32;`|Make an internal call and get back the return value.|Yes. But panics if call_input.amount is nonzero|
|Transfer|`fn transfer(transfer_input_ptr: *const [u8; 40]);`|Transfer funds to another account.|No|

### Internal command triggers (network commands)

Commands deferred using the functions described here are executed in FIFO order. If any command fails, the call itself must fail too.

This set of functions is in flux and set to change. Contract developers take care about depending on these functions.

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Defer create deposit|`fn defer_create_deposit(create_deposit_input_ptr: *const [u8], create_deposit_input_len: u32);`|Defer the execution of a create deposit command.|No|
|Defer set deposit settings|`fn defer_set_deposit_settings(set_deposit_settings_input_ptr: *const [u8], set_deposit_settings_input_len: u32);`|Defer the execution of a set deposit settings command.|No|
|Defer top up deposit|`fn defer_top_up_deposit(top_up_deposit_input_ptr: *const [u8], top_up_deposit_input_len: u32);`|Defer the execution of top up deposit command.|No|
|Defer withdraw deposit|`fn defer_withdraw_deposit(withdraw_deposit_input_ptr: *const [u8], withdraw_deposit_input_len: u32);`|Defer the execution of a withdraw deposit command.|No|
|Defer stake deposit|`fn defer_stake_deposit(stake_deposit_input_ptr: *const: [u8], stake_deposit_input_len: u32);`|Defer the execution of a stake deposit command.|No|
|Defer unstake deposit|`fn defer_unstake_deposit(unstake_deposit_input_ptr: *const [u8], unstake_deposit_input_len: u32);`|Defer the execution of an unstake deposit command.|No|

### Return value and log

Returning a value or pushing a log is charged immediately and as if the return value or the log will become included in a receipt. This is even, for example, if `return_value` is called inside an internal call. 

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|Return value|`fn return_value(return_val_ptr: *const [u8], return_val_len: u32);`|Return a value. If this function had already been called previously in the current call, this replaces the previous return value.|Yes|
|Log|`fn _log(log_ptr: *const [u8], log_len: u32);`|Push a log to the call command's logs.|Yes|

### Cryptographic operations

|Name|Signature|Description|In view calls?|
|---|---|---|---|
|SHA256|`fn sha256(msg_ptr: *const [u8], msg_len: u32, digest_ptr_ptr: *mut *const CryptoHash);`|Compute the SHA256 hash over a given message.|Yes|
|Keccak256|`fn keccak256(msg_ptr: *const [u8], msg_len: u32, digest_ptr_ptr: u32);`|Compute the Keccak256 hash over a given message.|Yes|
|RIPEMD160|`fn ripemd160(msg_ptr: *const [u8], msg_len: u32, digest_ptr_ptr: u32);`|Compute the RIPEMD160 hash over a given message.|Yes|
|Verify Ed25519 signature|`fn verify_ed25519_signature(msg_ptr: *const [u8], msg_len: u32, sig_ptr: *const [u8], addr_ptr: *const [u8]) -> u32;`|Verify an Ed25519 signature. Returns 1 if it is, and 0 otherwise.|Yes|

## Guest Functions

|Name|Signature|Description|
|---|---|---|
|Entrypoint|`fn entrypoint()`|The function called by the host in all contract call contexts.|
|Alloc|`fn alloc(len: u32) -> *mut u8`|Allocate a contiguous segment of memory of a specified length, and return a pointer to the beginning of the segment.|

## Appendix: Illegal  Opcodes

|Opcode family|Opcodes|
|---|---|
|Floating point operations|F32Eq, F32Lt, F32Const, F64Eq, F64Lt, F64Const, F32x4ConvertI32x4U, I32x4TruncSatF64x2SZero|
|SIMD instructions|I8x16GtU, I8x16LtS, I16x8GeS, I16x8GtU, I32x4MinU, I32x4MaxS, I64x2Sub, I64x2Mul, V128Const, V128Load8x8S|
|Atomic memory instructions|AtomicFence, I32AtomicLoad, I64AtomicRmwAdd, I32AtomicRmw8XchgU|
