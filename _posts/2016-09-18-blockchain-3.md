---
layout: post
title: 区块链技术 3 - fabric ledger
date: 2016-09-18 22:05:20 +08:00
tags: blockchain
---

## 0

fabric将区块链数据封装为`Ledger`，底层数据存储使用了`RocksDB`。

## DB

数据存储接口的封装位于`fabric/core/db/db.go`。

`RocksDB`数据库的位置通过`peer.fileSystemPath`配置，主要的ColumnFamily有五个：

1. BlockchainCF，存储区块链
2. StateCF，存储状态
3. StateDeltaCF，存储临时状态
4. IndexesCF，存储区块链的索引
5. PersistCF，存储consensu等状态信息

## Ledger

区块链接口封装在`fabric/core/ledger`包下。

主要的数据结构：

```go

/////////////////////////////////////////
//账本，单例
type Ledger struct {
	blockchain *blockchain
	state      *state.State
	currentID  interface{}
}

/////////////////////////////////////////
//区块链
type blockchain struct {
	size               uint64
	previousBlockHash  []byte
	// 索引，也就是读写IndexesCF
	indexer            blockchainIndexer
	// 最后处理的区块
	lastProcessedBlock *lastProcessedBlock
}

/////////////////////////////////////////
//状态
type State struct {
	// 默认buckettree
	stateImpl             statemgmt.HashableState
	stateDelta            *statemgmt.StateDelta
	currentTxStateDelta   *statemgmt.StateDelta
	currentTxID           string
	txStateDeltaHash      map[string][]byte
	updateStateImpl       bool
	historyStateDeltaSize uint64
}
```

主要的接口：

```go
// BeginTxBatch - gets invoked when next round of transaction-batch execution begins
// 通过ledger.currentID是否为nil判断
func (ledger *Ledger) BeginTxBatch(id interface{}) error 
```

```go
// CommitTxBatch - gets invoked when the current transaction-batch needs to be committed
// This function returns successfully iff the transactions details and state changes (that
// may have happened during execution of this transaction-batch) have been committed to permanent storage
func (ledger *Ledger) CommitTxBatch(id interface{}, transactions []*protos.Transaction, transactionResults []*protos.TransactionResult, metadata []byte) error 

//调用blockchain.addPersistenceChangesForNewBlock写数据库
//BlockchainCF blockNumber: blockBytes
//BlockchainCF blockCountKey("blockCount"): blockNumber+1

//调用blockchain.indexer.createIndexes写数据库
//IndexesCF blockHash: blockNumber
//IndexesCF tx.Txid: (blockNumber, txIndex)
//IndexesCF (address, blockNumber): txsIndexes

//调用state.AddChangesForPersistence写数据库
//StateCF dataKey: value
//StateCF bucketKey: marshal
//StateDeltaCF blockNumber: serializedStateDelta
```

```go
// RollbackTxBatch - Discards all the state changes that may have taken place during the execution of
// current transaction-batch
func (ledger *Ledger) RollbackTxBatch(id interface{}) error 

```

```go
// 通过state.currentTxID判断Tx状态
// TxBegin - Marks the begin of a new transaction in the ongoing batch
func (ledger *Ledger) TxBegin(txID string) 

// TxFinished - Marks the finish of the on-going transaction.
// If txSuccessful is false, the state changes made by the transaction are discarded
func (ledger *Ledger) TxFinished(txID string, txSuccessful bool) 

//如果成功，调用state.stateDelta.ApplyChanges更新stateDelta
```

```go
// GetState get state for chaincodeID and key. If committed is false, this first looks in memory
// and if missing, pulls from db.  If committed is true, this pulls from the db only.
func (ledger *Ledger) GetState(chaincodeID string, key string, committed bool) ([]byte, error) 
// 查数据库 StateCF (bucketNumber, chaincodeID, key)

// SetState sets state to given value for chaincodeID and key. Does not immideatly writes to DB
func (ledger *Ledger) SetState(chaincodeID string, key string, value []byte) error 

// DeleteState tracks the deletion of state for chaincodeID and key. Does not immediately writes to DB
func (ledger *Ledger) DeleteState(chaincodeID string, key string) error 
```

```go
// ApplyStateDelta applies a state delta to the current state. This is an
// in memory change only. You must call ledger.CommitStateDelta to persist
// the change to the DB.
// This should only be used as part of state synchronization. State deltas
// can be retrieved from another peer though the Ledger.GetStateDelta function
// or by creating state deltas with keys retrieved from
// Ledger.GetStateSnapshot(). For an example, see TestSetRawState in
// ledger_test.go
// Note that there is no order checking in this function and it is up to
// the caller to ensure that deltas are applied in the correct order.
// For example, if you are currently at block 8 and call this function
// with a delta retrieved from Ledger.GetStateDelta(10), you would now
// be in a bad state because you did not apply the delta for block 9.
// It's possible to roll the state forwards or backwards using
// stateDelta.RollBackwards. By default, a delta retrieved for block 3 can
// be used to roll forwards from state at block 2 to state at block 3. If
// stateDelta.RollBackwards=false, the delta retrieved for block 3 can be
// used to roll backwards from the state at block 3 to the state at block 2.
func (ledger *Ledger) ApplyStateDelta(id interface{}, delta *statemgmt.StateDelta) error 

// CommitStateDelta will commit the state delta passed to ledger.ApplyStateDelta
// to the DB
func (ledger *Ledger) CommitStateDelta(id interface{}) error {

//StateCF (bucketNumber, chaincodeID, key): value
//StateCF bucketKey: bucketNode
```

```go
// VerifyChain will verify the integrity of the blockchain. This is accomplished
// by ensuring that the previous block hash stored in each block matches
// the actual hash of the previous block in the chain. The return value is the
// block number of lowest block in the range which can be verified as valid.
// The first block is assumed to be valid, and an error is only returned if the
// first block does not exist, or some other sort of irrecoverable ledger error
// such as the first block failing to hash is encountered.
// For example, if VerifyChain(0, 99) is called and previous hash values stored
// in blocks 8, 32, and 42 do not match the actual hashes of respective previous
// block 42 would be the return value from this function.
// highBlock is the high block in the chain to include in verification. If you
// wish to verify the entire chain, use ledger.GetBlockchainSize() - 1.
// lowBlock is the low block in the chain to include in verification. If
// you wish to verify the entire chain, use 0 for the genesis block.
func (ledger *Ledger) VerifyChain(highBlock, lowBlock uint64) (uint64, error) 
```

## blockchain接口

跟区块链直接相关的接口主要有三个：

1. BeginTxBatch
2. CommitTxBatch
3. RollbackTxBatch

consensus模块做完一致性逻辑后调用这些接口完成写区块。`consensus/helper/helper.go`对这些接口又做了一次封装。

`Ledger`结构体中的`currentID`用于标记一次写区块的事务。

`CommitTxBatch`的过程主要包括：

1. `ledger.state.GetHash()`计算状态的散列值。`state.updateStateImpl`标记是否需要将`state.stateDelta`更新到`state.stateImpl`。
2. `ledger.blockchain.addPersistenceChangesForNewBlock`构建block，通过`writeBatch`写数据库。
3. `ledger.state.AddChangesForPersistence`写StateCF和StateDeltaCF。
4. 如果数据库写成功，`currentID`设为`nil`，清空`StateDelta`，递增`blockchain.size`。
5. 发送各种events。

## state接口

跟状态相关的接口主要有五个：

1. TxBegin
2. TxFinished
3. GetState
4. SetState
6. DeleteState

在执行交易`exectransaction.go`的过程中会调用这些接口对`state`进行修改。

`State`结构体中的`currentTxID`用于标记一次交易的事务。

如果执行成功，那么在`TxFinished`过程中会将`currentTxStateDelta`应用到`stateDelta`。

对状态的增删改查方法主要是供chaincode中的`shim.ChaincodeStubInterface`调用。

`GetState`依次查找`state.currentTxStateDelta`、`state.stateDelta`，最后通过`state.stateImpl.Get`查数据库。

`SetState`将kv赋值到`currentTxStateDelta`，而不是直接写数据库。
