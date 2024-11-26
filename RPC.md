# Remote Procedure Calls

|Revision no.|
|---|
|0|

This chapter specifies the ParallelChain Remote Procedure Call (RPC) API, an HTTP API that Fullnodes make available to clients. 

This chapter is organized into four sections. The first section, [Calling RPCs](#calling-rpcs), spells out the general properties of the API, including chiefly how it is implemented over the HTTP protocol. The next three sections then lists all of the RPCs available in API, grouping them into three categories:
1. [Transaction RPCs](#transaction-rpcs): RPCs for querying transactions and transaction-related information such as receipts, as well as for submitting transactions.
2. [Block RPCs](#block-rpcs): RPCs for querying blocks and block-related information, such as block headers.
3. [State RPCs](#state-rpcs): RPCs for querying the world state, e.g., for account balances, keys in storage tries, or information in the network account.

## Calling RPCs 

### HTTP parameters

A ParallelChain RPC call begins with a client sending an HTTP request over to a Fullnode and ends with that Fullnode sending over an HTTP response to the client in return. This HTTP request-response cycle must abide by the following parameters:

|Parameter|Value|
|---|---|
|Route|Each RPC offered by a Fullnode is reachable at a URL of the form "http://{host}/{shared_path}/{rpc_name}": <ul><li>Host: the domain name or the IP address that the Fullnode is accessible at.</li><li> Shared Path: an optional path under which each RPC endpoint directly sit. Useful if the host serves multiple HTTP APIs to separate them under different namespaces.</li><li> RPC Name: the name of the specific RPC.</li></ul><br/>For example, "http://example.com/fullnode_rpcs/submit_transaction" is a valid RPC route, and if a Fullnode serves the "submit_transaction" RPC on said route, the "block" RPC should be available on "http://example.com/fullnode_rpcs/block".|
|HTTP Method|Always "POST".|
|Request Content-Type|Always "application/octet-stream".|
|Response Status Code|We make the deliberate choice not to rely on HTTP status codes to report error cases to clients, as we find HTTP status codes restrictive and often misleading. Instead, we report errors in [response payloads](#request-and-response-payloads).<br/><br/>Therefore, the protocol assigns meaning only to two specific status codes:<ul><li>400 Bad Request: returned if the server cannot deserialize the request's body.</li><li>200 OK: returned in all other cases.</li></ul><br/>Implementations may choose to return other HTTP status codes (e.g., to report errors related to lower levels of the networking stack), but these have no protocol specified meaning.|
|Message Body|See [Requests and responses](#requests-and-responses).|

### Requests and responses

ParallelChain RPC calls operate based on a request-response cycle, with an RPC request data structure being carried on the HTTP request, and an RPC response data structure carried on the HTTP response.

RPC requests and responses are strongly typed, and each RPC endpoint has its own request and response types (which are specified for each endpoint later in the chapter). 

In order to be transmitted over HTTP, RPC requests and responses are to be serialized using [**Borsh**](http://borsh.io/), and placed inside the Body of an HTTP request or HTTP response, respectively. Each message body should contain the serialization of one RPC request, or of one RPC response, and no other bytes.

## Transaction RPCs

### submit_transaction 

Try to insert a Transaction into the Mempool. If the submitted transaction is successfully inserted into the target Fullnode's Mempool, it will broadcast the transaction to other Fullnodes in the P2P network.

#### Request

```rust
struct SubmitTransactionRequest {
    /// The transaction to be inserted into the mempool.  
    transaction: Transaction,
}
```

#### Response

```rust
struct SubmitTransactionResponse {
    /// The reason why the submitted transaction was not added to the mempool.
    error: Option<SubmitTransactionError>,
}
```

Where `SubmitTransactionError`:
```rust
enum SubmitTransactionError {
    /// The submitted transaction's Nonce (`transaction.nonce`) is either "too small" or "too large".
    ///
    /// What "too small" or "too large" means exactly is up to implementations, but the following general
    /// guidelines apply:
    /// 1. Too small: if `transaction.nonce` is less-than or equal to `transaction.signer`'s nonce in the
    ///    committed world state, then implementations should reject `transaction`, since it is impossible
    ///    at this point for `transaction` to be committed.
    /// 2. Too large: if `transaction.nonce` is *much* bigger than `transaction.signer`'s nonce in the
    ///    committed world state, then implementations can choose to reject `transaction`, on the grounds
    ///    that the Fullnode will have to wait too long before the transaction can be committed, and it
    ///    would rather use the precious space in its local mempool to store transactions that are more likely
    ///    to be committed in the near term.
    /// 3. Existing nonce: if the local mempool already contains a transaction with the same nonce and signer
    ///    as the `transaction`, the Fullnode can choose to reject `transaction`, e.g., if the existing
    ///    transaction has a higher Priority Fee than the submitted transaction.
    UnacceptableNonce,
    
    /// The Fullnode's local Mempool does not have the capacity to include the submitted transaction.
    MempoolFull,

    /// The Fullnode rejected the submitted transaction for reasons other than what is explicitly enumerated
    /// by the protocol.
    Other,
}
```

### transaction

Query a *committed* Transaction by its `transaction_hash`, returning not only the matching transaction but also the `block_hash` of the block that includes the transaction, the transaction's position within that block, and optionally, its receipt.

#### Request

```rust
struct TransactionRequest {    
    /// Hash of the transaction to be queried.
    transaction_hash: CryptoHash,

    /// Whether or not the queried transaction's Receipt should be included in the response.
    include_receipt: bool,
}
```

#### Response

```rust
struct TransactionResponse {
    /// The transaction with the specified `transaction_hash`, if it exists and is already committed.
    ///
    /// # Optionality of the rest of 
    ///
    /// If `transaction` is `None`, then the rest of `TransactionResponse`s fields will also `None`. 
    /// Likewise if `transaction` is `Some`, then the rest of the response fields will also be `Some`
    /// (`receipt` will only be `Some` if `include_receipt` is set to `true in the request).
    transaction: Option<Transaction>,

    /// The receipt of the returned transaction.
    receipt: Option<Receipt>,

    /// The Hash of the committed block that contains the returned transaction.
    block_hash: Option<CryptoHash>,

    /// The index of the returned transaction in the committed block's Transactions vector.
    position: Option<u32>,
}
```

### transaction_position

Query the `block_hash` of the committed Block that includes the Transaction identified by `transaction_hash`, as well as the `position` of the transaction within said block.

#### Request

```rust
struct TransactionPositionRequest {
    /// Hash of the transaction whose position is to be queried.
    transaction_hash: CryptoHash,
}
```

#### Response

Note: [^1].

```rust
struct TransactionPositionResponse {
    /// See Note: [1] above. 
    transaction_hash: Option<CryptoHash>,

    /// If the requested `transaction_hash` matches a transaction in a committed block, then, the block's 
    /// hash. Else `None`.
    block_hash: Option<CryptoHash>,

    /// If `block_hash` is `Some`, then , the index of the queried transaction in the committed block's
    /// Transactions vector.
    position: Option<u32>,
}
```

[^1]: This field is an erratum and has no semantic meaning. Fullnodes should include this field in their `TransactionPositionResponse`s, but clients should always ignore its value.

### receipt

Query the Receipt of the *committed* Transaction identified by `transaction_hash`, returning also the hash of the block that said transaction is included in, and the position of the transaction in the block.

#### Request

```rust
struct ReceiptRequest {
    /// Hash of the transaction whose receipt is to be queried.
    transaction_hash: CryptoHash,
}
```

#### Response

```rust
struct ReceiptResponse {
    /// Hash of the transaction whose receipt was queried (this takes the same value as the
    /// `transaction_hash` field of the request corresponding to this response).
    transaction_hash: CryptoHash,

    /// The queried transaction's receipt, if said transaction has been included in a committed block.
    receipt: Option<Receipt>,

    /// Hash of the committed block in which the queried transaction was included in.
    block_hash: Option<CryptoHash>,

    /// Index of the queried transaction in the Transactions vector of the committed block in which it is
    /// included. This equals the index of the transaction's receipt in the same block's Receipts vector.
    position: Option<u32>,
}
```

## Block RPCs

### block

Query a Block (pending or committed) by its `block_hash`.

#### Request

```rust
struct BlockRequest {
    /// Hash of the block to be queried.
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockResponse {
    /// The block with the queried `block_hash`. This block may be pending, or committed.
    block: Option<Block>,
}
```

### block_header

Get the Block Header of a Block (pending or committed) by its `block_hash`.

#### Request

```rust
struct BlockHeaderRequest {
    /// Hash of the block whose header is to be queried.
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockHeaderResponse {
    /// The header of the block with the queried `block_hash`. The block may be pending, or committed.
    block_header: Option<BlockHeader>,
}
```

### block_height_by_hash

Get the height of the committed Block identified by `block_hash`.

#### Request

```rust
struct BlockHeightByHashRequest {
    /// Hash of the block whose height will be queried.
    block_hash: CryptoHash,
}
```

#### Response

```rust
struct BlockHeightByHashResponse {
    /// Hash of the block whose height was queried (this takes the same value as the `block_hash` field
    /// of the request corresponding to this response).
    block_hash: CryptoHash,

    /// Height of the queried block. `None` if the block does not exist or has not been committed yet.
    block_height: Option<BlockHeight>,
}
```

### block_hash_by_height

Get the hash of the committed Block at the specified `block_height`.

#### Request

```rust
struct BlockHashByHeightRequest {
    /// Height of the block whose hash will be queried.
    block_height: BlockHeight,
}
```

#### Response

```rust
struct BlockHashByHeightResponse {
    /// Height of the block whose hash was queried (this takes the same value as the `block_height` field
    /// of the request corresponding to this response).
    block_height: BlockHeight,

    /// Hash of the queried block. `None` if the block does not exist or has not been committed yet.
    block_hash: Option<CryptoHash>,
}
```

### highest_committed_block

Get the hash of the current highest committed Block (the committed block with the highest block height). 

#### Request

Empty.

#### Response

```rust
struct HighestCommittedBlockResponse {
    /// Hash of the current highest committed block. `None` if no blocks have been committed yet.
    block_hash: Option<CryptoHash>,
}
```

## State RPCs

State RPCs return multiple entities in the world state in a single response. This allows clients to get a consistent snapshot of the world state in a single call.

Every response structure includes the hash of the highest committed block when the snapshot is taken.

Some of the following RPCs' response structures reference types unique to this document. These are specified in [types featuring in state-related RPC responses](#types-featuring-in-state-related-rpc-responses).

### state

Get the [fields](World%20State.md#Account-fields) of a set of `accounts` (choosing whether or not to include their contract code(s)), and/or a set of `storage_keys`.

#### Request

```rust
struct StateRequest {
    /// The set of accounts whose fields (Nonce, Balance, CBI version, and Storage hash) will be queried.
    accounts: HashSet<PublicAddress>,
    
    /// Whether the Contract field should also be queried for the requested accounts.
    include_contracts: bool,
    
    /// A set of accounts (`PublicAddress`) mapped to the set of storage keys (`HashSet<Vec<u8>>`) that 
    /// should be queried for each account.
    storage_keys: HashMap<PublicAddress, HashSet<Vec<u8>>>,
}
```

#### Response

```rust
struct StateResponse {
    /// The queried accounts.
    accounts: HashMap<PublicAddress, Account>,

    /// The queried storage tuples.
    storage_tuples: HashMap<PublicAddress, HashMap<Vec<u8>, Vec<u8>>>,

    /// Hash of the highest committed block at the point in time in which the world state was queried. 
    block_hash: CryptoHash,
}
```

See type definition: [`Account`](#account).

### validator_sets

Get information about the Previous, Current, and Next Validator Sets, optionally including the stakes delegated to them.  

#### Request

```rust
struct ValidatorSetsRequest {
    /// Whether to query the previous validator set (PVS).
    include_prev: bool,

    /// Whether to include the delegators included in the PVS.
    include_prev_delegators: bool,

    /// Whether to query the current validator set (CVS).
    include_curr: bool,

    /// Whether to include the delegators included in the CVS.
    include_curr_delegators: bool,

    /// Whether to query the next validator set (NVS).
    include_next: bool,

    /// Whether to include the delegators included in the NVS.
    include_next_delegators: bool,
}
```

#### Response

```rust
struct ValidatorSetsResponse {
    /// The previous validator set. 
    ///
    /// This is `None` if the request specifies `!include_prev`, `Some(None)` if the request specifies
    /// `include_prev` but the current epoch number is 0, and `Some(Some(pvs))` otherwise.
    prev_validator_set: Option<Option<ValidatorSet>>,

    /// The current validator set. 
    ///
    /// This is `None` if the request specifies `!include_prev`.
    curr_validator_set: Option<ValidatorSet>,

    /// The next validator set.
    ///
    /// This is `None` if the request specifies `!include_prev`.
    next_validator_set: Option<ValidatorSet>,

    /// Hash of the highest committed block at the point in time in which the PVS, CVS, and NVS were
    /// queried. 
    block_hash: CryptoHash,
}
```

### pools

Get information about the pools operated by a given set of `operators`, optionally including the stakes currently contained in each pool.

#### Request

```rust
struct PoolsRequest {
    /// The operators whose pools will be queried.
    operators: HashSet<PublicAddress>,

    /// Whether or not to include the stakes currently contained in each pool in the response.
    include_stakes: bool,
}
```

#### Response

```rust
struct PoolsResponse {
    /// The queried Pools.
    ///
    /// If the requested operator does not currently operate a pool, the value mapped to it will be `None`.
    pools: HashMap<PublicAddress, Option<Pool>>,
    
    /// Hash of the highest committed block at the point in time in which the requested pools were 
    /// queried. 
    block_hash: CryptoHash,
}
```

See type definition: [`Pool`](#pool).

### deposits

Get information about the deposits associated with a given set of `(operator, owner)` pairs. 

Each `(operator, owner)` pair asks to query information about the deposit owned by `owner` which is assigned to the pool operated by `operator`.


#### Request

```rust
struct DepositsRequest {
    /// The `(operator, owner)` pairs that identify the deposits to be queried.
    deposits: HashSet<(PublicAddress, PublicAddress)>,
}
```

#### Response

```rust
struct DepositsResponse {
    /// The queried deposits.
    ///
    /// If the requested `(owner, operator)` pair is not currently associated with a deposit, it will be 
    /// mapped to `None`.
    deposits: HashMap<(PublicAddress, PublicAddress), Option<Deposit>>,

    /// Hash of the highest committed block at the point in time in which the requested deposits were
    /// queried.
    block_hash: CryptoHash,
}
```

See type definition: [`Deposit`](#deposit).

### stakes

Get information about the stakes associated with a given set of `(operator, owner)` pairs in the current [Next Validator Set](World%20State.md#network-account-storage-fields).

Each `(operator, owner)` pair asks to query information about the stake owned by `owner` which is currently staked to the pool operated by `operator`.

For information about the stakes in the current Previous Validator Set and Current Validator Set, consider calling the [validator_sets](#validator_sets) RPC with the `include_prev_delegators` or `include_curr_delegators` flags enabled, respectively.

#### Request

```rust
struct StakesRequest {
    /// The `(operator, owner)` pairs that identify the stakes to be queried.
    stakes: HashSet<(PublicAddress, PublicAddress)>,
}
```

#### Response

```rust
struct StakesResponse {
    /// The queried stakes.
    ///
    /// If the requested `(operator, owner)` pair is not currently associated with a stake, it will be 
    /// mapped to `None`. 
    stakes: HashMap<(PublicAddress, PublicAddress), Option<Stake>>,

    /// Hash of the highest committed block at the point in time in which the requested stakes were
    /// queried.
    block_hash: CryptoHash,
}
```

See type definition: [`Stake`](#stake).

### view

[View-call](Contracts.md#view-calls) a given `method` in a `target` contract and get the [Command Receipt](Blockchain.md#receipt-v1) describing the results of the execution.

#### Request

```rust
struct ViewRequest {
    /// Address of the contract that will be view-called.
    target: PublicAddress,

    /// The name of the method to view-call.
    method: Vec<u8>,

    /// The arguments to pass to the view call.
    arguments: Option<Vec<Vec<u8>>>,
}
```

#### Response

```rust
struct ViewResponse {
    /// Command receipt describing the results of the view call execution.
    receipt: CommandReceipt,
}
```

## Types featuring in state-related RPC responses

### Account

```rust
enum Account {
    WithContract(AccountWithContract),
    WithoutContract(AccountWithoutContract),
}
```

### Account With Contract

```rust
struct AccountWithContract {
    nonce: Nonce,
    balance: Balance,
    contract: Option<Vec<u8>>, 
    cbi_version: Option<CBIVersion>,
    storage_hash: Option<CryptoHash>,
}
```

### Account Without Contract

```rust
struct AccountWithoutContract {
    nonce: Nonce,
    balance: Balance,
    cbi_version: Option<CBIVersion>,
    storage_hash: Option<CryptoHash>,
}
```

### Validator Set

```rust
enum ValidatorSet {
    WithDelegators(Vec<PoolWithDelegators>),
    WithoutDelegators(Vec<PoolWithoutDelegators>),
}
```

### Pool 

```rust
enum Pool {
    WithStakes(PoolWithDelegators),
    WithoutStakes(PoolWithoutDelegators),
}
```

### Pool With Delegators

```rust
struct PoolWithDelegators {
    operator: PublicAddress,
    power: Balance,
    commission_rate: u8, 
    operator_stake: Option<Stake>,
    delegated_stakes: Vec<Stake>,
}
```

### Pool Without Delegators

```rust
struct PoolWithoutDelegators {
    operator: PublicAddress,
    power: Balance,
    commission_rate: u8, 
    operator_stake: Stake,
}
```

### Deposit

```rust
struct Deposit {
    owner: PublicAddress,
    balance: u64,
    auto_stake_rewards: bool,
}

```

### Stake

```rust
struct Stake {
    owner: PublicAddress,
    power: Balance,
}
```
