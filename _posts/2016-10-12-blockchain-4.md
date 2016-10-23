---
layout: post
title: 区块链技术 4 - fabric chaincode
date: 2016-10-23 22:05:20 +08:00
tags: blockchain
---

## 0

fabric将区块链之上的业务逻辑封装在chaincode中执行。

## Chaincode

chaincode的示例可以从`examples/chaincode/go`目录下找到。

简单来说，业务代码需要实现接口：

```go
type Chaincode interface {
	// Init is called during Deploy transaction after the container has been
	// established, allowing the chaincode to initialize its internal data
	Init(stub ChaincodeStubInterface, function string, args []string) ([]byte, error)

	// Invoke is called for every Invoke transactions. The chaincode may change
	// its state variables
	Invoke(stub ChaincodeStubInterface, function string, args []string) ([]byte, error)

	// Query is called for Query transactions. The chaincode may only read
	// (but not modify) its state variables and return the result
	Query(stub ChaincodeStubInterface, function string, args []string) ([]byte, error)
}
```

其中`Init`方法在deploy时执行，`Invoke`提供调用的入口，基本所有的业务都是在该方法中实现，`Query`提供查询接口。

这三个方法通过`ChaincodeStubInterface`提供的接口调用`peer`的服务。

## chaincode 部署

在`dev`模式下，chaincode需要自己注册name，而生产环境中则使用`-p`参数指定代码路径，`peer`服务将会完成注册，并且返回name。

```plain
CORE_PEER_ADDRESS=172.17.0.2:7051 peer chaincode deploy -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -c '{"Function":"init", "Args": ["a","100", "b", "200"]}'
```

以上述命令为例：

peer客户端解析参数，构建`request`，然后通过`devopsClient.Deploy`调用`peer`服务。

```go
type ChaincodeSpec struct {
	// -l 参数
	Type                 ChaincodeSpec_Type   `protobuf:"varint,1,opt,name=type,enum=protos.ChaincodeSpec_Type" json:"type,omitempty"`
	// -n -p 参数
	ChaincodeID          *ChaincodeID         `protobuf:"bytes,2,opt,name=chaincodeID" json:"chaincodeID,omitempty"`
	// -c 参数
	CtorMsg              *ChaincodeInput      `protobuf:"bytes,3,opt,name=ctorMsg" json:"ctorMsg,omitempty"`
	Timeout              int32                `protobuf:"varint,4,opt,name=timeout" json:"timeout,omitempty"`
	// -u 如果securityEnable=true，赋值loginToken_{$u}
	SecureContext        string               `protobuf:"bytes,5,opt,name=secureContext" json:"secureContext,omitempty"`
	// 如果privacyEnable=true，根据配置赋值
	ConfidentialityLevel ConfidentialityLevel `protobuf:"varint,6,opt,name=confidentialityLevel,enum=protos.ConfidentialityLevel" json:"confidentialityLevel,omitempty"`
	// 自定义 peer命令行没有提供参数
	Metadata             []byte               `protobuf:"bytes,7,opt,name=metadata,proto3" json:"metadata,omitempty"`
	// -a 参数
	Attributes           []string             `protobuf:"bytes,8,rep,name=attributes" json:"attributes,omitempty"`
}
```

服务端执行`Deploy`主要流程如下：

a) getChaincodeBytes
  
获取代码，根据路径http下载到`$GOPATH/_usercode_`或者本地`$GOPATH`
  
spec.ChaincodeID.Name = hash(CtorMsg + [all file content])
  
返回`ChaincodeDeploymentSpec`
  
```go
  type ChaincodeDeploymentSpec struct {
	ChaincodeSpec *ChaincodeSpec `protobuf:"bytes,1,opt,name=chaincodeSpec" json:"chaincodeSpec,omitempty"`
	// Controls when the chaincode becomes executable.
	EffectiveDate *google_protobuf.Timestamp                   `protobuf:"bytes,2,opt,name=effectiveDate" json:"effectiveDate,omitempty"`
	CodePackage   []byte                                       `protobuf:"bytes,3,opt,name=codePackage,proto3" json:"codePackage,omitempty"`
	ExecEnv       ChaincodeDeploymentSpec_ExecutionEnvironment `protobuf:"varint,4,opt,name=execEnv,enum=protos.ChaincodeDeploymentSpec_ExecutionEnvironment" json:"execEnv,omitempty"`
}
```
  
其中`CodePackage`是所有源文件打的tar包，还有一个Dockerfile
  
```plain
FROM hyperledger/fabric-ccenv:$(ARCH)-$(PROJECT_VERSION)
COPY src $GOPATH/src
WORKDIR $GOPATH
RUN go install %s && cp src/github.com/hyperledger/fabric/peer/core.yaml $GOPATH/bin && mv $GOPATH/bin/%s $GOPATH/bin/%s"
```
  
b) 如果启动安全，那么通过crypto对transaction进行签名；否则直接构建Transaction

c) `d.coord.ExecuteTransaction(tx)`将transaction交给consensus模块，在pbft中，消息进入`eer.manager.Queue()`就返回结果

d) 如果进入consensus模块成功，`Deploy`就返回`ChaincodeID.Name`，但是这并不表示chaincode已经部署成功。

consensus模块的流程在此略过，最终会调用`chaincode.ExecuteTransactions`执行具体的chaincode，主要流程如下：

a) 如果启动安全，那么首先解密transaction
b) `chain.Deploy`部署Docker镜像
c) `chain.Launch`启动镜像，镜像启动之后调用peer服务注册，然后peer服务调用chaincode容器执行`init`方法。

在代码实现中chaincode服务两部分形式化为状态机：

shim状态机

![](http://ofenxpygt.bkt.clouddn.com/shim.png)

ChaincodeSupport状态机

![](http://ofenxpygt.bkt.clouddn.com/chaincode.png)

## chaincode 调用

invoke和query执行过程类似，参数处理完之后通过`d.coord.ExecuteTransaction(tx)`将transaction交给consensus模块。

query直接同步调用`chaincode.Execute(`执行query方法返回结构；invoke与deploy类似是异步执行的。两者具体执行流程参考上一节中的两个状态机，在此就不详细描述了。
