---
layout: post
title: 区块链技术 2 - fabric overview
date: 2016-08-26 22:05:20 +08:00
tags: blockchain
---

## 0

[Fabric](https://gerrit.hyperledger.org/r/#/admin/projects/fabric)是IBM牵头发起的区块链平台，隶属于Linux基金会的[Hyperledger](https://www.hyperledger.org)项目。

Fabric的[文档](http://hyperledger-fabric.readthedocs.io/en/latest/)相对比较详细，开发环境很容易搭建。（当时得有VPN，不然某些资源下载不到）

本文将简要介绍Fabric的整体代码结构。

## 整体结构

```shell
.
├── LICENSE
├── Makefile
├── README.md
├── TravisCI_Readme.md
├── bddtests #行为驱动测试
├── consensus #一致性逻辑
│   ├── consensus.go #定义各种接口
│   ├── controller
│   ├── executor
│   ├── helper
│   ├── noops #用于开发，实际上没有处理一致性的逻辑
│   ├── pbft #拜占庭容错
│   └── util
├── core #各模块核心逻辑
│   ├── admin.go #管理接口，查看staus等操作
│   ├── admin_test.go
│   ├── chaincode #chaincode逻辑
│   ├── comm
│   ├── config
│   ├── config.go
│   ├── container #chaincode执行容器
│   ├── crypto #与membersrvc交互，证书、加解密相关逻辑
│   ├── db
│   ├── devops.go #devops接口，chaincode的deploy、invok、query都从这里开始
│   ├── devops_test.go
│   ├── discovery #发现网络中peers
│   ├── fsm.go
│   ├── fsm_test.go
│   ├── ledger #账本的逻辑，也就是blockchain和world state
│   ├── peer #节点的逻辑，处理各种消息
│   ├── rest #rest接口
│   ├── system_chaincode #系统chaincode
│   └── util
├── devenv #vagrant开发环境
├── docs
├── events
├── examples #例子
├── flogging #日志，设置等级
├── gotools
├── images
├── membersrvc #成员管理逻辑
├── metadata
│   └── metadata.go
├── mkdocs.yml
├── peer #peer命令入口，以及处理逻辑
│   ├── chaincode #peer chaincode命令入口
│   ├── common
│   ├── core.yaml
│   ├── main.go
│   ├── main_test.go
│   ├── network #peer node命令入口
│   ├── node #peer node命令入口
│   ├── util
│   └── version
├── protos #各种protos文件
├── pub
├── report.xml
├── scripts
├── sdk
├── settings.gradle
├── tools
│   ├── busywork #测试工具
│   └── dbutility #db工具
└── vendor #依赖包
```

## corbra & viper

[corbra](https://github.com/spf13/cobra)是一个处理命令行交互的框架

[viper](https://github.com/spf13/viper)是一个配置管理框架

fabric使用了上述两个框架处理命令行交互和参数配置，基本上看着代码就能理解流程，具体的细节在此不描述。

（值得一提的是这两个项目出自同一个开发者——[spf13](https://github.com/spf13)）

## gRPC

[gRPC-Go](https://github.com/grpc/grpc-go)是Google推出的RPC框架

Fabric使用了上述框架做RPC，代码中出自的`pb.`代码都是与此有关。

## peer node start

```shell
cd $GOPATH/src/github.com/hyperledger/fabric
make peer
peer node start --peer-chaincodedev
```

这是开发者文档中出现的第一条命令，也就是启动节点，笔者将从此开始串一下代码流程。 

### 1

`peer/main.go`

`viper`读`core.yaml`，`corbra`处理命令行参数

`peer/node/start.go`

经过`corbra`最后调到了`serve(args)`方法

### 2

`serve`方法中启动各种服务：

1. pb.RegisterChaincodeSupportServer #chaincode支持服务，standalone模式
2. pb.RegisterPeerServer #peer节点服务
3. pb.RegisterAdminServer #管理服务
4. pb.RegisterDevopsServer #devops服务
5. pb.RegisterOpenchainServer #openchain服务

注册上述grpc服务到`peer.listenAddress`端口（默认7051）

1. pb.RegisterEventsServer #只有vp需要创建EventHub服务

注册上述grpc服务到`peer.validator.events.address`（默认7053）

具体的服务接口查看`protos`文件夹下对应的`proto`文件比较清楚。

### 3

启动db，这里使用的是`rocksdb`，存储blockchain、state等

`db.Start()`

### 4

接下来再深入看一下创建Validating Peer的流程

```go
logger.Debug("Running as validating peer - making genesis block if needed")
makeGenesisError := genesis.MakeGenesis()
if makeGenesisError != nil {
	return makeGenesisError
}
logger.Debugf("Running as validating peer - installing consensus %s",
	viper.GetString("peer.validator.consensus"))

peerServer, err = peer.NewPeerWithEngine(secHelperFunc, helper.GetEngine)
```

首先创建创世块，调用ledger的接口

```go
func MakeGenesis() error {
	once.Do(func() {
		ledger, err := ledger.GetLedger()
		if err != nil {
			makeGenesisError = err
			return
		}

		if ledger.GetBlockchainSize() == 0 {
			genesisLogger.Info("Creating genesis block.")
			if makeGenesisError = ledger.BeginTxBatch(0); makeGenesisError == nil {
				makeGenesisError = ledger.CommitTxBatch(0, nil, nil, nil)
			}
		}
	})
	return makeGenesisError
}
```

然后新建peer，engine实际上就是consensus，然后与peers的`PeerServer`建立连接

```go
func NewPeerWithEngine(secHelperFunc func() crypto.Peer, engFactory EngineFactory) (peer *Impl, err error) {
	peer = new(Impl)
	peerNodes := peer.initDiscovery()

	peer.handlerMap = &handlerMap{m: make(map[pb.PeerID]MessageHandler)}

	peer.isValidator = ValidatorEnabled()
	peer.secHelper = secHelperFunc()

	// Install security object for peer
	if SecurityEnabled() {
		if peer.secHelper == nil {
			return nil, fmt.Errorf("Security helper not provided")
		}
	}

	// Initialize the ledger before the engine, as consensus may want to begin interrogating the ledger immediately
	ledgerPtr, err := ledger.GetLedger()
	if err != nil {
		return nil, fmt.Errorf("Error constructing NewPeerWithHandler: %s", err)
	}
	peer.ledgerWrapper = &ledgerWrapper{ledger: ledgerPtr}

	peer.engine, err = engFactory(peer)
	if err != nil {
		return nil, err
	}
	peer.handlerFactory = peer.engine.GetHandlerFactory()
	if peer.handlerFactory == nil {
		return nil, errors.New("Cannot supply nil handler factory")
	}

	peer.chatWithSomePeers(peerNodes)
	return peer, nil

}
```

## peer chaincode deploy

通过`corbra`找到`peer/chaincode/deploy.go`中的`chaincodeDeploy`方法。

调用`DevopsServer`的`Deploy`方法，在该方法中主要调用了peer的`ExecuteTransaction`方法，也就说将部署这一操作当做区块链中的一笔“交易”，“交易”上链了就代表部署成功。

最后调用consensus引擎中的`ProcessTransactionMsg`实现。

在noops中就是广播transaction消息为consensus消息，然后通过`processTransactions`方法执行chaincode、提交ledger。

## membersrvc

如果peer启动安全模式，那么还需启动`membersrvc`

`membersrvc/server.go`

`viper`读取`membersrvc.yaml`配置文件

启动下述gRPC服务，实际上就是一系列CA：

1. pb.RegisterACAPServer(srv, &ACAP{aca}) #attribute CA
2. pb.RegisterECAPServer(srv, &ECAP{eca}) #enroll CA 公共接口
3. pb.RegisterECAAServer(srv, &ECAA{eca}) #enroll CA 管理接口
4. pb.RegisterTCAPServer(srv, &TCAP{tca}) #transaction CA 公共接口
5. pb.RegisterTCAAServer(srv, &TCAA{tca}) #transaction CA 管理结构
6. pb.RegisterTLSCAPServer(srv, &TLSCAP{tlsca}) #tls CA 公共接口
7. pb.RegisterTLSCAAServer(srv, &TLSCAA{tlsca}) #tls CA 管理接口

上述服务注册到端口`server.port`（默认7054）

具体的服务接口查看`protos`文件夹下的`ca.proto`文件比较清楚。

## peer network login

如果peer启动安全模式，用户需要首先执行登陆操作。

通过`corbra`找到`peer/network/login.go`中的`networkLogin`方法。

首先检查`peer.fileSystemPath`路径下的token文件，判断用户是否已经登陆；如果没有登陆，连接`DevopsServer`调用`Login`方法。

在`DevopsServer`中调用了`crypto.RegisterClient`方法，连接`membersrvc`服务生成证书，并且保存在本地`peer.fileSystemPath`路径下的数据库文件中。
