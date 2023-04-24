# Remote Procedure Calls

|Revision no.|
|---|
|0|

This document specifies the ParallelChain RPC API, an HTTP API that Fullnodes make available to clients. 

We spell out the general properties of the API, and then list the available RPCs. With each RPC, we define a request structure and a response structure. 

For readability, we categorize RPCs into [transaction-related RPCs](#transaction-rpcs), [block-related RPCs](#block-rpcs), and [state-related RPCs](#state-rpcs).


## General properties

1. Procedures are reachable over HTTP at a URL suffixed by the procedure’s name.
2. All HTTP requests should be POST.
3. Request and response structures are serialized using Borsh and carried in HTTP message bodies. 
4. Procedures are “strongly typed”. If a procedure receives a request that cannot be deserialized, it will send back a response with an empty body and a Bad Request (400) status code.
5. Conversely, if a procedure receives a request that can be deserialized, the response it sends back must have an OK (200) status code. This is even if, for example, something was “not found” (e.g., a block was not found with a specified hash), or a transaction was not added to the mempool. Error statuses are reported in response structures.
6. All requests and OK responses should have a content-type of "application/octet-stream". This is so that we can easily extend the RPC API in the future to support other serialization schemes, e.g., JSON using "application/json".

## Transaction RPCs

### submit_transaction 

Submit a transaction to the mempool.

#### Request

```rust
struct SubmitTransactionRequest {
    transaction: Transaction,
}
```

#### Response

```rust
struct SubmitTransactionResponse {
    error: Option<SubmitTransactionError>,
}
```

Where `SubmitTransactionError`:
```rust
enum SubmitTransactionError {
    UnacceptableNonce,
    MempoolFull,
    Other,
}
```

### transaction

Get a transaction and optionally its receipt.

#### Request

```rust
struct TransactionRequest {    
    transaction_hash: CryptoHash,
    include_receipt: bool,
}
```

#### Response

```rust
struct TransactionResponse {
    transaction: Option<Transaction>,
    receipt: Option<Receipt>,
    block_hash: Option<CryptoHash>,
    position: Option<u32>,
}
```

### transaction_position

Find out where a transaction is in the blockchain.

#### Request

```rust
struct TransactionPositionRequest {
    transaction_hash: CryptoHash,
}
```

#### Response

```rust
struct TransactionPositionResponse {
    transaction_hash: Option<CryptoHash>,
    block_hash: Option<CryptoHash>,
    position: Option<u32>,
}
```

### receipt

Get a transaction's receipt.

#### Request

```rust
struct ReceiptRequest {    
    transaction_hash: CryptoHash,
}
```

#### Response

```rust
struct ReceiptResponse {
    transaction_hash: CryptoHash,
    receipt: Option<Receipt>,
    block_hash: Option<CryptoHash>,
    position: Option<u32>,
}
```

## Block RPCs

### block

Get a block by its block hash.

#### Request

```rust
struct BlockRequest {
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockResponse {
    block: Option<Block>,
}
```

### block_header

Get a block header by its block hash.

#### Request

```rust
struct BlockHeaderRequest {
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockHeaderResponse {
    block_header: Option<BlockHeader>,
}
```

### block_height_by_hash

Get the height of the block with a given block hash. 

#### Request

```rust
struct BlockHeightByHashRequest {
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockHeightByHashResponse {
    block_hash: CryptoHash,
    block_height: Option<BlockHeight>,
}
```

### block_hash_by_height

Get the hash of a block at a given height.

#### Request

```rust
struct BlockHashByHeightRequest {
    block_height: BlockHeight,
}
```

#### Response

```rust
struct BlockHashByHeightResponse {
    block_height: BlockHeight,
    block_hash: Option<CryptoHash>,
}
```

### highest_committed_block

Get the hash of the highest committed block. 

#### Request

None.

#### Response

```rust
struct HighestCommittedBlockResponse {
    block_hash: Option<CryptoHash>,
}
```

## State RPCs

State RPCs return multiple entities in the world state in a single response. This allows clients to get a consistent snapshot of the world state in a single call.

Every response structure includes the hash of the highest committed block when the snapshot is taken.

Some of the following RPCs' response structures reference types unique to this document. These are specified in [types referenced in state RPC responses](#types-referenced-in-state-rpc-responses).

### state

Get the state of a set of accounts (optionally including their contract code), and/or a set of storage tuples.

#### Request

```rust
struct StateRequest {
    accounts: HashSet<PublicAddress>,
    include_contracts: bool,
    storage_keys: HashMap<PublicAddress, HashSet<Vec<u8>>>,
}
```

#### Response

```rust
struct StateResponse {
    accounts: HashMap<PublicAddress, Account>,
    storage_tuples: HashMap<PublicAddress, HashMap<Vec<u8>, Vec<u8>>>,
    block_hash: CryptoHash,
}
```

### validator_sets

Get the previous, current, and next validator sets, optionally including the stakes delegated to them.  

#### Request

```rust
struct ValidatorSetsRequest {
    include_prev: bool,
    include_prev_delegators: bool,
    include_curr: bool,
    include_curr_delegators: bool,
    include_next: bool,
    include_next_delegators: bool,
}
```

#### Response

```rust
struct ValidatorSetsResponse {
    // The inner Option is None if we are at Epoch 0.
    prev_validator_set: Option<Option<ValidatorSet>>,
    curr_validator_set: Option<ValidatorSet>,
    next_validator_set: Option<ValidatorSet>,
    block_hash: CryptoHash,
}
```

### pools

Get a set of pools.

#### Request

```rust
struct PoolsRequest {
    operators: HashSet<Operator>,
    include_stakes: bool,
}
```

#### Response

```rust
struct PoolsResponse {
    pools: HashMap<Operator, Option<Pool>>,
    block_hash: CryptoHash,
}
```

### deposits

Get a set of deposits.

#### Request

```rust
struct DepositsRequest {
    stakes: HashSet<(Operator, Owner)>,
}
```

#### Response

```rust
struct DepositsResponse {
    deposits: HashMap<(Operator, Owner), Option<Deposit>>,
    block_hash: CryptoHash,
}
```

### stakes

Get a set of deposits.

#### Request

```rust
struct StakesRequest {
    stakes: HashSet<(Operator, Owner)>,
}
```

#### Response

```rust
struct StakesResponse {
    stakes: HashMap<(Operator, Owner), Option<Stake>>,
    block_hash: CryptoHash,
}
```

### view

Call a method in a contract in a read-only way.

#### Request

```rust
struct ViewRequest {
    target: PublicAddress,
    method: Vec<u8>,
    arguments: Option<Vec<Vec<u8>>>,
}
```

#### Response

```rust
struct ViewResponse {
    receipt: CommandReceipt,
}
```

## Types referenced in state RPC responses

### Account-related types
```rust
enum Account {
    WithContract(AccountWithContract),
    WithoutContract(AccountWithoutContract),
}

struct AccountWithContract {
    nonce: Nonce,
    balance: Balance,
    contract: Option<Vec<u8>>, 
    cbi_version: Option<CBIVersion>,
    storage_hash: Option<CryptoHash>,
}

struct AccountWithoutContract {
    nonce: Nonce,
    balance: Balance,
    cbi_version: Option<CBIVersion>,
    storage_hash: Option<CryptoHash>,
}
```

### Staking-related types 

```rust
enum ValidatorSet {
    WithDelegators(Vec<PoolWithDelegators>),
    WithoutDelegators(Vec<PoolWithoutDelegators>),
}

type Operator = PublicAddress;
type Owner = PublicAddress;

struct PoolWithDelegators {
    operator: PublicAddress,
    power: Balance,
    commission_rate: u8, 
    operator_stake: Option<Stake>,
    delegated_stakes: Vec<Stake>,
}

struct PoolWithoutDelegators {
    operator: PublicAddress,
    power: Balance,
    commission_rate: u8, 
    operator_stake: Stake,
}

struct Deposit {
    owner: PublicAddress,
    balance: u64,
    auto_stake_rewards: bool,
}

struct Stake {
    owner: PublicAddress,
    power: Balance,
}

enum Pool {
    WithStakes(PoolWithStakes),
    WithoutStakes(PoolWithoutStakes),
}
```
