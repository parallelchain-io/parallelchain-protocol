# Updates

|Revision no.|
|---|
|0|

This document describes:
* How the ParallelChain protocol and its individual aspects are updated.
* How different versions of the protocol are identified using version numbers and names.
* How updates are rolled out incrementally so that client software have time to be updated.

## Aims

Public blockchain systems like Mainnet are difficult to update, and for two major reasons: 
1. They consist of multitudes of independent participants that only weakly coordinate with one another and may take a long time to update to new implementations.
2. They usually commit to maintaining chain history as far back in time as possible and potentially forever.

Because of the first reason, protocol versions should maintain some level of backward compatibility. Because of the second reason, new protocol versions should avoid being bloated by the history of previous versions.

ParallelChain protocol updates aim to achieve a good balance between the two desiderata.

## Version identification

### Protocol versions

Each version of the protocol is identified by a version number of the form 0.x.y before "stabilization" (fully decentralized deployment of Mainnet), and the form x.y. after.

Here, x is called the major version, and y the minor version.

The major version is increased if a release will probably break existing replica or client implementations and maintaining compatibility will require more than changing a few configuration variables.

The minor version is increased if a release only adds features, i.e., without removing or changing existing features in a way that breaks existing replica or client implementations. Breaking changes that only require existing implementations to change configuration variables (e.g., changing the cost of an operation, or changing the target block time) may also be included as part of a minor version update.

A protocol update is considered to “break” implementations if:
* Existing replicas will no longer be able to produce or validate new blocks.
* Existing clients will no longer be able to publish new transactions into the blockchain,
* Or RPCs used by existing clients will no longer be available.

### CBI versions

CBI versions are also identified using major and minor versions. The major version of a contract is specified in its account state in the “CBI version” field.

The major version of the CBI is increased if:
* The contract validity rules are changed,
* The signature or behavior of an existing host function is changed,
* Or the signature or expected behavior of an existing guest function is changed.

The minor version is increased for all other changes, including if a new host function is added. Changes to gas metering do not cause the CBI version to be updated.	

### Revision numbers

To help readers identify changes between versions of the protocol, documents in this protocol are given revision numbers. These start at 0 and are incremented every time the section receives a substantive change (i.e., more than just a change in wording).

## Incremental rollout

Every major version of the protocol is associated with a specific block height (“start height”). These are set to be some time in the future after the new version of the protocol specification is published so that developers have time to change their implementations. Breaking changes to the protocol take effect on the start height. 

For changes to the chain, the runtime, or the world state, this means that the block at the start height will be the first to reflect the breaking changes.

For changes to the RPC, this means that deprecated RPCs stop being available to clients after a block at the start height is committed.

### Support for old RPC endpoints

RPCs are strongly typed. After a named RPC is introduced, the type of its request and response structures will never change. 

If the functionality that an RPC serves changes such that new request and response types are required, then a new RPC will be created with the same name but a different version suffix. 

As a concrete example, currently, the RPC for getting a block is called “block”. When a type-incompatible change to block is introduced in a new version of the protocol, a new RPC will be made available called “block/v2” which returns an enum in its response. The enum’s variants are the old version of block, and the new version. Clients will still be able to call the old “block” RPC until the new version’s start height.

### Support for old CBI versions

Until we introduce a mechanism for updating contract code and their CBI version, contracts targeting all existing CBI versions will be callable in the foreseeable future.


## Appendix: relationships between protocol aspects

As described in the introduction, the ParallelChain protocol is composed of 8 clearly-delineated aspects. These aspects can be changed independently and correspond each to a separate document in this specification.

Even though these aspects are clearly delineated, they do depend on one another, and a change in one aspect may force a change in another aspect.

In the diagram below, a link points out that a change in one aspect may cause a change in the other aspect. The following subsections discuss the kinds of changes that a change in one aspect may cause in other aspects:

![Protocol aspects relationships graph](assets/Aspects%20relationships.drawio.png)

### Consensus

New fields may be added to HotStuff-rs blocks. These fields will have to be added in ParallelChain block headers as well.

### Chain

New transaction commands may be added, or existing ones obsoleted. The execution of these commands will have to be specified in the runtime.

Changes to block and transaction types will also force the creation of new RPCs, since RPCs are strongly typed.

### Runtime

New transaction commands as well as changes in the executions of existing staking and network commands may require changes in the data structures stored in the network account storage. 

### World State

Protocol-defined data structures in the network account storage are served in the RPC. If these data structures are changed, new RPCs must be created.

### Gas

If the costs of operations are changed, runtimes will charge for transactions differently.

Conversely, new operations may be added to the runtime in new or existing commands. These may need to be assigned gas costs.

### CBI

A new version of the CBI may be added. The runtime needs to be able to call contracts targeting this new version. These contracts also need to be callable through the RPC’s view procedure.

Conversely, new commands may be added to the runtime. If contracts are to trigger these commands internally, new host functions need to be added to the CBI in a new version.

