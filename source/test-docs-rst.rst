# 权限体系整体设计文档RST

|  版本号  |  编写人  |  日期  |  变更内容  |
| --- | --- | --- | --- |
|  1.0  |  刘XX  |  2024.02  |  创建初始版本  |
|   |   |   |   |

<!-- **引言** -->

## 1. 引言

### 1.1 背景

联盟链Hyperchain需要一个权限体系，对链的操作权限、节点的权限等进行划分。以便于更好的管理，提供安全的服务。

### 1.2 目标

为hyperchain提供权限管理功能，包括链级权限、节点级权限。并根据这些权限提供对hyperchian的管理服务。

### 1.3 术语

|  英文  |  中文  |  含义  |
| --- | --- | --- |
|  CNS(Contract Namespace)  |  合约命名服务  |  合约命名服务  |
|  VP(Validate Peer)  |  验证节点，即共识节点  |  共识节点  |
|  CA(Certificate Authority)  |  证书颁发机构  |  签发证书、认证证书、管理已颁发证书的机构  |
|  RBAC(Role-Based Access Control)  |  基于角色的权限控制  |   |

<!-- **需求** -->
## 2 需求

### 2.1 功能性需求

#### 2.1.1 链级权限管理

提供链级权限管理机制，可通过此机制对一些链级功能进行管理。

针对链级权限管理机制，包含以下功能：

1.  可设置链级角色，便于进行基于角色的权限管理。其中**内置**了链级管理员、合约管理员、vp节点这几个角色。
    
2.  链级管理员可通过提案投票的方式对链级角色进行增加、删除，对链级角色进行账户授权和回收。
    

基于上述机制，提供如下链级功能管理：

1.  **VP节点的增删管理**：
    
    1.  分布式ca模式下，（未成功新增的节点无法与其他节点正常建立逻辑连接）
        
        1.  由链级管理员或vp节点发起节点新增或删除的提案请求
            
        2.  链级管理员对节点增删的提案请求进行投票
            
        3.  投票通过则由节点增删的提案请求发起者执行提案，完成节点增删；（若投票不通过或超时则节点增删失败）
            
    2.  中心化ca模式下，
        
        1.  新增节点时：由具有ca颁发证书的新节点构造交易，老节点帮其转发交易，完成当前新节点的新增
            
        2.  删除节点时：由链级管理员发送交易直接删除节点，并吊销被删除节点使用的证书
            
    3.  无ca模式下，
        
        1.  新增节点时：由新节点构造交易，老节点帮其转发交易，完成当前新节点的新增
            
        2.  删除节点时：由用户发送交易直接删除节点
            
2.  **链级配置管理**：链级管理员可通过提案投票的方式对链级配置进行修改。
    

可通过提案修改的链级配置项如下：

1.  设置proposal.timeout的值，即提案超时时间（默认超时时间为5分钟，即最短超时时间，当设置当超时时间小于最短超时时间时，会设置为最短超时时间）
    
2.  设置proposal.threshold的值，即提案的投票阈值（默认值为链级管理员总个数，另外阈值的范围为1-链级管理员总数，不能小于1，不能大于链级管理员总数）
    
3.  设置proposal.contract.vote.enable的值，即是否开启通过投票管理合约生命周期，默认关闭
    
4.  设置proposal.contract.vote.threshold的值，即合约生命周期管理提案的投票阈值（默认值为合约管理员总个数，另外阈值的范围为1-合约管理员总数，不能小于1，不能大于合约管理员总数）
    
5.  设置filter.enable的值，即是否开启交易拦截过滤器
    
6.  设置filter.rules的值，即交易拦截过滤规则
    
7.  设置consensus.algo值，即使用的共识算法（目前只在链上记录，不生效，即不使用链上记录的共识算法，且不支持修改。目前支持的值为`RBFT` 、`RAFT` 、`NoxBFT` 、`SOLO` ）
    

在新链第一次启动时，会将配置的genesis账户初始化为链级管理员和合约管理员，并将提案阈值以及合约生命周期管理的投票阈值默认设为对应管理员的总数、提案的超时时间默认设为5分钟（当设置的超时时间小于5分钟时，会设置为5分钟），即创建提案的交易打包时间+5分钟则为提案超时时间

1.  **合约权限管理**：是基于链级角色，对账户进行链上的合约（也可以是转账交易的to）访问权限进行管理。合约权限管理的开关以及拦截规则 (TX>2.3)
    

合约权限管理是基于链级角色，为账户分配不同的**链级角色**，然后使用链级配置管理，设置基于角色的交易拦截规则以及交易拦截开关。当开关打开时，设置了交易拦截规则后，交易在被执行之前会按照这个规则检验交易发送者是否有权限发送此交易。

拦截规则包含以下字段：

*   allowAnyone : bool 是否允许任何人访问
    
*   authorizedRoles : \[\]string 允许访问的角色列表
    
*   forbiddenRoles : \[\]string 禁止访问的角色列表
    
*   id : int 规则id
    
*   name : string 规则名称
    
*   to :  \[\]string 规则限制的交易地址（可以是合约地址、或账户地址，目前支持通配符\*）
    
*   vm ：\[\]string 规则限制的虚拟机类型（目前支持的是evm 、 hvm、bvm三种，支持通配符\*）
    

当同一交易地址被多个拦截规则匹配时，会按照**id更小**当规则进行校验。对于上述拦截规则中包含的字段，其优先级为forbiddenRoles > allowAnyone > authorizedRoles。对于没有拦截规则限制的交易地址，则不进行校验拦截。

例如，当某笔交易的to在多个规则中被添加了拦截规则时，会取出id更小的规则进行校验，先验证交易发起者是否含有被禁止的角色，有则校验不通过；再校验是否允许任何人访问，是则校验通过；最后校验交易发起者是否含有允许访问的角色列表，是则校验通过，否则校验不通过。

2.  合约生命周期管理(contract\_proposal)：合约生命周期管理包括合约的部署、升级、冻结、解冻以及销毁操作。现合约生命周期可通过两种方式进行管理，一种是不投票的方式，使用默认对方式管理合约；一种是提案投票的方式，需要提出申请，合约管理员审核通过后才能进行合约管理操作。
    

当前使用投票方式管理合约还是不投票的方式管理合约取决于配置项`proposal.contract.vote.enable` 的值，当值为true时只能使用投票管理合约，当值为false时只能使用不投票的方式管理合约（默认方式）。如果采用投票的方式管理合约，需要多少赞同票才通过则通过配置项`proposal.contract.vote.threshold` 的值确定。

3.  合约命名服务（CNS），为一个合约地址绑定一个别名。目前一个合约地址只能有一个合约命名，一个合约命名只能绑定一个合约地址。其操作步骤如下：
    

1.  由一个合约管理员或合约部署者发送交易，创建一个合约命名类型的提案（注：为内置合约绑定别名只能由**合约管理员**发送交易创建提案，为其他普通合约绑定别名只能由**合约部署者**发送交易创建提案）；
    
2.  等待合约管理员对提案进行投票：合约管理员发送交易，对这个提案投票，当收集到足够的同意票数后提案进入等待执行状态；
    
3.  发送交易创建这个提案的发起者发送交易，执行这个投票通过的提案，完成合约地址与合约命名的绑定。
    

合约地址与合约命名绑定之后，可以通过合约命名执行合约，也可以根据合约命名查询相应数据。

### 2.1.2 节点级权限管理

提供节点级权限管理机制，可通过此机制对节点服务进行管理。

针对节点级权限管理机制，包含以下功能：

1.  可设置节点级角色，便于进行基于角色的权限管理。注意，此节点级角色是指这个节点在某一ns下生效的角色，并不是对节点所在的所有ns都生效的角色。
    

基于上述机制，提供如下节点级功能管理：

1.  接口权限管理：用于对节点对查询接口（不包含发送交易的接口）的权限进行管理，是节点级的接口权限管理。
    

如果某一节点针对某一namespace设置了该节点的接口访问规则，则只对此节点中对应对namespace生效，对于此节点对其他namespace和其他节点是不生效的。

支持的操作：

1.  设置开关（接口权限管理是节点级的，默认关闭）
    
2.  角色管理（为账户设置节点级的角色，角色的设定没有限制，不需要先创建角色再设置，直接为账户设置角色即可）
    
3.  规则管理（基于角色来设置接口的管理规则，当开关开启时，规则生效）
    

### 2.2 非功能性需求

<!-- **整体设计** -->
## 3 整体设计

### 3.1 整体架构图

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/eYVOLwRZBVymqpz2/img/6fbb5009-63bf-44d9-81ec-351dc30fd205.png)

### 3.2 模块划分

1.  rpc：jsonrpc、grpc两种接收api请求的方式，都需要添加接口权限拦截器。api请求调用前，先通过拦截器检验是否有调用权限。
    
2.  interceptor：增加实现接口权限管理拦截器。当接口权限管理开关打开时，根据调用信息、拦截规则验证是否有调用权限。
    
3.  api：接收交易时，检查是否为配置交易，若是则标记给配置交易；检查交易发送方是否有向交易接收方发送交易的权限
    
4.  consensus：
    
    1.  对配置交易进行识别，收到配置交易后对配置交易进行单独打包，并停止打包新的区块；
        
    2.  待配置交易执行完成后，变更epoch，直接出ckp，抛出事件通知外部模块；
        
    3.  监听配置交易修改的配置项，并根据需要进行相应的变更，比如共识算法切换、共识版本升级、共识节点管理等
        
5.  execute：
    
    1.  执行区块时：将合约命名转换为合约地址；检出内置合约修改的链级配置；执行完后通知config中的cm更新链级配置
        
    2.  替换账本时：替换完账本后，通知config中的cm更新链级配置
        
6.  epochMgr: 维护监听ckp事件的模块，并在出ckp时通知到对应模块进行相应变更
    
7.  bvm：内置虚拟机，管理内置合约，通过内置合约提供链级权限机制、合约管理、合约命名服务、链级配置管理等功能。当对链级配置进行修改时，使用特定的接口对其进行更改。
    
8.  statedb：缓存记录链级配置的修改，提供签出功能。
    
9.  config：提供链级配置项管理；配置文件同步变更；管理与区块同步的链级配置的更新及其监听模块，在配置发生变更时通知相应模块
    
10.  namespace：启动时 创建gensis区块，在genesis块中写入创世链级配置；触发同步账本中的链级配置。
    
11.  rbac：
    
    1.  针对链级：提供合约调用权限验证功能；监听与区块同步的链级角色相关配置，并根据变化进行缓存等更新。
        
    2.  针对节点级：提供节点级权限管理机制；提供接口调用权限验证功能。
        
12.  其他：
    
    1.  vpMgr: 非分布式ca模式下，协助新增的新节点发送新增交易，完成节点新增。
        
    2.  txgen：采用节点公私钥构造交易
        
    3.  cns：提供合约地址与合约命名的转换功能
        

**详细设计**

链级权限管理机制：通过内置合约，提案投票、账户角色管理完成链级权限管理。主要通过bvm完成

[https://alidocs.dingtalk.com/i/nodes/y20BglGWO20gPqGDTP41jdGK8A7depqY](https://alidocs.dingtalk.com/i/nodes/y20BglGWO20gPqGDTP41jdGK8A7depqY)

vp节点管理：

[三方应用: https://alidocs.dingtalk.com/i/nodes/R4GpnMqJzGl07kO9TEbADjw38Ke0xjE3](https://alidocs.dingtalk.com/i/nodes/R4GpnMqJzGl07kO9TEbADjw38Ke0xjE3)

链级配置管理：

[https://alidocs.dingtalk.com/i/nodes/1OQX0akWmxLjawB4sP4ngkDw8GlDd3mE?utm\_scene=team\_space&sideCollapsed=true&iframeQuery=utm\_source%253Dportal%2526utm\_medium%253Dportal\_new\_tab\_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288](https://alidocs.dingtalk.com/i/nodes/1OQX0akWmxLjawB4sP4ngkDw8GlDd3mE?utm_scene=team_space&sideCollapsed=true&iframeQuery=utm_source%253Dportal%2526utm_medium%253Dportal_new_tab_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288)

合约权限管理：

[https://alidocs.dingtalk.com/i/nodes/PwkYGxZV3ZpA0vLkc2d5RRDpWAgozOKL?utm\_scene=team\_space&sideCollapsed=true&iframeQuery=utm\_source%253Dportal%2526utm\_medium%253Dportal\_new\_tab\_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288](https://alidocs.dingtalk.com/i/nodes/PwkYGxZV3ZpA0vLkc2d5RRDpWAgozOKL?utm_scene=team_space&sideCollapsed=true&iframeQuery=utm_source%253Dportal%2526utm_medium%253Dportal_new_tab_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288)

合约生命周期管理：

[https://alidocs.dingtalk.com/i/nodes/Exel2BLV5zPAmDgKsQB1ojZ9Jgk9rpMq](https://alidocs.dingtalk.com/i/nodes/Exel2BLV5zPAmDgKsQB1ojZ9Jgk9rpMq)

合约命名服务：

[https://alidocs.dingtalk.com/i/nodes/0eMKjyp8134A1KMjioLOKMyoVxAZB1Gv?utm\_scene=team\_space&sideCollapsed=true&iframeQuery=utm\_source%253Dportal%2526utm\_medium%253Dportal\_new\_tab\_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288](https://alidocs.dingtalk.com/i/nodes/0eMKjyp8134A1KMjioLOKMyoVxAZB1Gv?utm_scene=team_space&sideCollapsed=true&iframeQuery=utm_source%253Dportal%2526utm_medium%253Dportal_new_tab_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288)

节点级权限管理机制及节点级接口权限管理：

[https://alidocs.dingtalk.com/i/nodes/PwkYGxZV3ZpA0vLkc2d5kK1EWAgozOKL?utm\_scene=team\_space&sideCollapsed=true&iframeQuery=utm\_source%253Dportal%2526utm\_medium%253Dportal\_new\_tab\_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288](https://alidocs.dingtalk.com/i/nodes/PwkYGxZV3ZpA0vLkc2d5kK1EWAgozOKL?utm_scene=team_space&sideCollapsed=true&iframeQuery=utm_source%253Dportal%2526utm_medium%253Dportal_new_tab_open&corpId=dinged56b58b141823d3ee0f45d8e4f7c288)

**接口设计**

监听与区块同步的链级配置的监听器：

    // Reloader is the interface should be implement by the structures,
    // which want to be informed to reload config in commit stages
    // while config items or related data they watch are changed by config tx
    type Reloader interface {
    
        // Reload will be called after writing block successfully
        // if the watched config items are changed
        Reload(newValue interface{}) error
    
        // NeedValue is need new value when call Reload method
        // if return true, call Reload method with newValue
        // if return false, call Reload method with nil
        NeedValue() bool
    }

监听ckp的监听器：

    // StableSubmitter is the interface should be implemented by the structures
    // which want to be informed to process stable checkpoint
    type StableSubmitter interface {
        // SubmitCheckpoint will be called when new stable checkpoint generated.
        // NOTE: only return error for module can not handle the fatal, the namespace
        // will be stopped immediately for each error returned.
        SubmitCheckpoint(checkpoint *protos.QuorumCheckpoint, config bool) error
    }
