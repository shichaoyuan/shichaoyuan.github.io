---
layout: post
title: 区块链技术 1 - 简介
date: 2016-08-01 22:05:20 +08:00
tags: blockchain
---

## 0

区块链技术理解起来非常简单，如下图所示：

```plain
+---------+       +---------+
| payload |       | payload |
| nonce   | ----> | nonce   | ----> ...
| preHash |       | preHash |
+---------+       +---------+
```

简单来说，收集一段时间内的数据，做为一个区块，计算该区块**满足条件**的hash值，然后就可以将该区块append到链上，并且记下前一个区块的hash值。如此这样，如果某人想要篡改历史数据，那么hash值就会改变，首先改变的hash值一般不会**满足条件**，即使碰巧满足条件，也需要继续篡改后续区块。

所谓的**满足条件**也就是共识机制，bitcoin使用的是PoW，但是由于其过于浪费计算资源（因而篡改的成本也比较高），所以近来又涌现了不少新的共识机制，比如PoS、PBFT。

本文简要介绍区块生成和上链的基本流程，示例代码来自[izqui/blockchain](https://github.com/izqui/blockchain)。

## ECDSA密钥

```go
type Keypair struct {
	Public  []byte `json:"public"`  // base58 (x y)
	Private []byte `json:"private"` // d (base58 encoded)
}

func GenerateNewKeypair() *Keypair {

	pk, _ := ecdsa.GenerateKey(elliptic.P224(), rand.Reader)

	b := bigJoin(KEY_SIZE, pk.PublicKey.X, pk.PublicKey.Y)

	public := base58.EncodeBig([]byte{}, b)
	private := base58.EncodeBig([]byte{}, pk.D)

	kp := Keypair{Public: public, Private: private}

	return &kp
}

```

当某个节点加入网络，首先生成ECDSA密钥，公钥做为该节点的标识，私钥用于签名。

## Blockchain数据结构

```go
//Transaction也就是节点每次的操作的信息
//在bitcoin的场景下就是转账事务
type Transaction struct {
	Header    TransactionHeader
	Signature []byte //signed(sha256(header))
	Payload   []byte
}

type TransactionHeader struct {
	From          []byte //发起节点的公钥
	To            []byte //目标节点的公钥
	Timestamp     uint32
	PayloadHash   []byte //sha256(payloadData)
	PayloadLength uint32 //payloadData长度
	Nonce         uint32 //Proof of work
}

type TransactionSlice []Transaction
```

```go
//block header
type BlockHeader struct {
	Origin     []byte //打包该块的节点的公钥
	PrevBlock  []byte //前一个block header的sha256
	MerkelRoot []byte //transaction的sha256
	Timestamp  uint32
	Nonce      uint32
}

//block
type Block struct {
	*BlockHeader
	Signature []byte //signed(sha256(header))
	*TransactionSlice
}
```

```go
type BlockSlice []Block

type Blockchain struct {
	CurrentBlock Block //当前时间区间接收到的transaction
	BlockSlice //所谓的区块链

	TransactionsQueue // 接收其他节点发来的transaction的chan
	BlocksQueue // 接收其他节点发来的block的chan
}
```

## 创建Transaction

```go
func CreateTransaction(txt string) *Transaction {
	t := NewTransaction(Core.Keypair.Public, nil, []byte(txt))
	// 循环遍历**满足条件**的Nonce
	t.Header.Nonce = t.GenerateNonce(TRANSACTION_POW)
	t.Signature = t.Sign(Core.Keypair)

	return t
}

func (t *Transaction) GenerateNonce(prefix []byte) uint32 {
	newT := t
	for {
		if CheckProofOfWork(prefix, newT.Hash()) {
			break
		}
		newT.Header.Nonce++
	}
	return newT.Header.Nonce
}

```

## PoW检查

```go
var (
	TRANSACTION_POW = helpers.ArrayOfBytes(TRANSACTION_POW_COMPLEXITY, POW_PREFIX)
	BLOCK_POW       = helpers.ArrayOfBytes(BLOCK_POW_COMPLEXITY, POW_PREFIX)

	TEST_TRANSACTION_POW = helpers.ArrayOfBytes(TEST_TRANSACTION_POW_COMPLEXITY, POW_PREFIX)
	TEST_BLOCK_POW       = helpers.ArrayOfBytes(TEST_BLOCK_POW_COMPLEXITY, POW_PREFIX)
)

//简单来说就是要使得hash的前几个byte与prefix相同
//POW_COMPLEXITY在bitcoin中有一套增加难度的算法[Difficulty](https://en.bitcoin.it/wiki/Difficulty)，
//这里就先不详细介绍了
func CheckProofOfWork(prefix []byte, hash []byte) bool {
	if len(prefix) > 0 {
		return reflect.DeepEqual(prefix, hash[:len(prefix)])
	}
	return true
}
```

## 接收Transaction

```go

		select {
		case tr := <-bl.TransactionsQueue:
			// 如果sig相同，表示已经收到
			if bl.CurrentBlock.TransactionSlice.Exists(*tr) {
				continue
			}
			// 验证payload hash、pow、sig
			if !tr.VerifyTransaction(TRANSACTION_POW) {
				fmt.Println("Recieved non valid transaction", tr)
				continue
			}
			// 有效的transaction添加到CurrentBlock
			bl.CurrentBlock.AddTransaction(tr)
			interruptBlockGen <- bl.CurrentBlock

			//Broadcast transaction to the network
			mes := NewMessage(MESSAGE_SEND_TRANSACTION)
			mes.Data, _ = tr.MarshalBinary()

			time.Sleep(300 * time.Millisecond)
			Core.Network.BroadcastQueue <- *mes

```

## 创建Block

```go
		block := <-interrupt
	loop:
		fmt.Println("Starting Proof of Work...")
		block.BlockHeader.MerkelRoot = block.GenerateMerkelRoot()
		block.BlockHeader.Nonce = 0
		block.BlockHeader.Timestamp = uint32(time.Now().Unix())

		for true {

			sleepTime := time.Nanosecond
			if block.TransactionSlice.Len() > 0 {

				if CheckProofOfWork(BLOCK_POW, block.Hash()) {

					block.Signature = block.Sign(Core.Keypair)
					bl.BlocksQueue <- block
					sleepTime = time.Hour * 24
					fmt.Println("Found Block!")

				} else {

					block.BlockHeader.Nonce += 1
				}

			} else {
				sleepTime = time.Hour * 24
				fmt.Println("No trans sleep")
			}

			select {
			case block = <-interrupt:
				goto loop
			case <-helpers.Timeout(sleepTime):
				continue
			}
		}

```

## 接收Block

```go
		case b := <-bl.BlocksQueue:
			// 如果sig相同，表示已经收到
			if bl.BlockSlice.Exists(b) {
				fmt.Println("block exists")
				continue
			}
			// 验证merkel、pow、sig
			if !b.VerifyBlock(BLOCK_POW) {
				fmt.Println("block verification fails")
				continue
			}

			if !reflect.DeepEqual(b.PrevBlock, bl.CurrentBlock.PrevBlock) {
				// I'm missing some blocks in the middle. Request'em.
				fmt.Println("Missing blocks in between")
			} else {

				fmt.Println("New block!", b.Hash())

				transDiff := TransactionSlice{}

				if !reflect.DeepEqual(b.BlockHeader.MerkelRoot, bl.CurrentBlock.MerkelRoot) {
					// Transactions are different
					fmt.Println("Transactions are different. finding diff")
					transDiff = DiffTransactionSlices(*bl.CurrentBlock.TransactionSlice, *b.TransactionSlice)
				}
				// 满足条件就上链
				bl.AddBlock(b)

				//Broadcast block and shit
				mes := NewMessage(MESSAGE_SEND_BLOCK)
				mes.Data, _ = b.MarshalBinary()
				Core.Network.BroadcastQueue <- *mes

				//New Block
				bl.CurrentBlock = bl.CreateNewBlock()
				bl.CurrentBlock.TransactionSlice = &transDiff

				interruptBlockGen <- bl.CurrentBlock
			}

```
