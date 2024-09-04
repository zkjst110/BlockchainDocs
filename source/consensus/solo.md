<head>

<meta name="referrer" content="no-referrer"/>
</head>

# SOLO对外介绍文档

> \*版本为功能版本，对于一期、二期需求，一期版本为1.0，二期为2.0

|  版本  |  日期  |  变更人员  |  变更原因  |  变更内容  |  备注  |
| --- | --- | --- | --- | --- | --- |
|  1.0  |  20240819  |  袁超  |  1.  初始文档       |  1.  创建文档       |   |

## 1 引言

### 1.1 编写目标

本文档旨在介绍 Solo 共识的相关功能、设计原理，不涉及内部实现细节。主要用于调研技术背景时可以通过本文档简单获取所需说明。

### 1.2 阅读对象

 本文档供其他模块开发同学以及项目规划、课题管理同学使用。

## 2 功能背景

### 2.1 现存问题

Hyperchian 平台支持多种共识协议，包括：RBFT、NoxBFT 和 DagBFT 。这些共识协议都需要多节点的参与，需要在平台上配置所有共识节点的网络地址、对端节点信息、服务端口等信息。这给部署和使用平台的人员增加了不少工作量，直接提高了 Hyperchain 的试用门槛。

### 2.2 解决方案

为了简化共识的配置信息及工作流程，Hyperchain 提供了单节点共识算法 **Solo** 便于用户试用与测试。

此外 Solo 共识还可以辅助 Hyperchain 其他模块的测试。由于 Solo 是单节点共识协议，屏蔽了网络延迟等不可控因素。使得开发人员可以专注于自己模块本身的开发和调试。

### 2.3 关键技术

Solo 协议运行时，不会和其它节点通信与达成共识，仅通过单个 Hyperchain 节点来完成交易的打包与区块的提交。

Solo 接收到新交易事件后，将其缓存在**内存交易池（mempool）**中。

Solo 协议会在启动时设置交易数量**打包阈值（batch size）以及定时器（timer）**，当交易池中交易的数量达到阈值，或是定时器超时后，Solo 会将这些交易（batch）打包成一个区块。

由于 Solo 协议不需要共识协商，Solo 在打包完区块后，立刻将区块提交给 Hyperchain 的 Executor 执行，**这个行为是异步的，Solo 并不会等待每一个区块执行完毕后才提交下一个区块**。

### 2.4 术语说明

|  英文  |  中文  |  含义  |
| --- | --- | --- |
|  transaction  |  交易  |  用户改变账户状态、调用智能合约的操作单元  |
|  batch  |  交易批次  |  包含一批交易的结构  |
|  block  |  区块  |  区块链上的共识单元，包含一组交易，以及交易执行后改变账本状态的摘要信息  |
|  event  |  事件  |  Hyperchain 中各个组件交互的结构单元  |
|  checkpoint  |  检查点  |  Hyperchain 中共识协议达成最终一致的确认结构  |

## 3 设计方案

### 3.1 整体框架

##### 3.1.1 整体架构图

Solo 的整体架构非常简洁，仅有一个内存交易池和 Solo 共识组件构成。

在运行过程中，Solo 负责与 EventHub、Executor、CheckpointMgr 交互。

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/2510bb4b-6cb9-4730-a18d-a9532bcca7cd.png)

*   Client：发送交易的客户端
    
*   EventHub：处理所有事件的事件中心
    
*   mempool：solo 的内存交易池
    
*   Consensus：共识协议管理模块
    
*   Solo：Solo 共识协议
    
*   Executor：区块执行模块
    
*   CheckpointMgr：检查点模块
    

### 3.2 模块设计

##### 3.2.1 内存交易池（mempool）

###### 3.2.1.1 概述

内存交易池（mempool）用于 Solo 共识协议的交易缓存，它仅仅用于临时存放交易，不会对交易做持久化，也不会对交易进行验证

###### 3.2.1.2 流程

**新增交易：**

向交易池中加入新的交易，如果新增的交易恰好能满足打包 batch 生成条件，则返回生成的 batch

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/7870fd68-dcaf-48ba-a714-2c4a7af65177.png)

**打包交易：**

将交易池中，未打包成 batch 的交易强制打包成 batch

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/c0d66b0e-9163-4107-b037-1c70f6dab43c.png)

**打包配置交易：**

Solo 协议会以特殊方式处理标记类型是**配置交易**的交易，将其**单独打包成一个 batch**

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/fe65d6ca-9055-4570-ab8a-2afe47ca0a1d.png)

##### 3.2.2 Solo 协议模块

###### 3.2.2.1 概述

Solo 协议模块实现了 Solo 共识所需要的所有逻辑

###### 3.2.2.2 流程

**接收交易**

Solo 监听来自客户端的交易，并将它们打包成 batch 提交给平台的执行模块

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/ec321add-78eb-443f-a319-55201399e09b.png)

**接收区块**

Solo 监听来自执行模块的区块执行结果，并生成检查点提交给平台的 CheckpointManager 模块

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/4maOgkv0orGKOWNX/img/beb23bee-9ced-4ff4-ad31-9b8379ff24d8.png)

### 3.3 接口设计

##### 3.3.1 用户接口

**GetNodeStates 获取 VP 节点状态**

```json
{
  "id": 1,
  "hostname": "node1",
  "hash": "34d299742260716bab353995fe98727004b5c27bde52489f61de093176e82088",
  "status": "NORMAL",
  "epoch": 1,
  "checkpoint": 16715,
  "blockHeight": 16715,
  "blockHash": "000000000000414bd74cdad2ec880ccec57febb7f5ac3c973a9129d849e37af6",
  "version": "undefined"
}
```

##### 3.3.2 外部接口

**Metrics 外部可监控指标**

|  名称  |  类型  |  说明  |
| --- | --- | --- |
|  incoming\_txs  |  Counter  |  Solo 处理所有交易的数量  |
|  duplicate\_txs  |  Counter  |  Solo 处理所有重复交易的数量  |
|  nonBatched\_txs  |  Gauge  |  交易池中未打包的交易数量  |
|  pending\_txs  |  Gauge  |  未进入到交易池的交易数量  |

##### 3.3.3 内部接口

**Start 启动 Solo 协议**

```go
Start(...) error
```

**Stop 停止 Solo 协议**

```go
Stop()
```

**Status 查看当前 Solo 协议的状态**

```go
Status() types.SystemStatus
```
```go
type SystemStatus struct {
	Status       uint64
	StatusDetail string
	Epoch        uint64
	ID           uint64
	Checkpoint   uint64
    ...
}
```

**Nodes 查看当前运行 Solo 的节点信息**

```go
Nodes() types.NodesList
```
```go
type NodeInfo struct {
	ID        uint64
	Namespace string
	Hash      string
	Hostname  string
	IsVP      bool
	IsPrimary bool
}
```

**Order 向 Solo 协议发送交易事件**

```go
Order(tx *events.NewTxEvent) error
```

### 3.4 相关配置项说明

|  配置项  |  说明  |  默认值  |
| --- | --- | --- |
|  BatchSize  |  每个 batch 包含交易的笔数  |  500  |
|  BatchMemLimit  |  是否限制每个 batch 的字节大小  |  false  |
|  BatchMaxMem  |  开启字节限制后，每个 batch 的字节大小限制  |  0  |
