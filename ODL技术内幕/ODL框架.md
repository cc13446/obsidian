## ODL 框架
### 架构设计
平台化，提出了业务抽象层`Service Abstraction Layer，SAL`的概念，把控制器分为南向协议插件、北向协议插件和SAL层。通过SAL层实现南向协议插件和多样化的业务应用的结构。

### SAL的演进
1. AD-SAL：分别定义南向和北向插件的API，并为两种接口做大量的适配编码，手工定义北向REST
2. MD-SAL：YANG语言作为数据及接口的建模语言，利用YangTools工具自动生成代码

### 设计目标
1. 来自不同供应商的不同物理和虚拟设备类型的互操作性
2. 提供网络流量从源到目的的可视性
3. 面向所有设备的通用的管理框架
4. 可根据用户需求塑造网络行为的可变成性
5. 基于策略的自动化

### 总体架构
ODL基于Karaf这个强大的OSGI容器来提供组件的部署和运行环境，把被控制和管理的网络抽象为一个以消息驱动的巨大的状态机，因此消息机制和状态保存机制就成了整体架构中最基础的服务设施。

引入YANG语言作为消息和状态数据的模型语言，通过其设计的基础服务设施称为模型驱动的服务抽象层，也就是`MD-SAL`。`MD-SAL`是整个架构的核心，为南向协议插件和业务应用提供了统一的服务调用接口。具体来说就是`RPC`，`Notification`，数据变更通知这三种消息机制，同时提供了DataStore来保存设备与业务的配置和状态。

![](ODL框架/Pasted%20image%2020220819220344.png)

MD-SAL典型的模块间调用与交互机制：
1. RPC提供了消费者和生产者之间一对一的调用路由机制
2. Notification提供了消息的订阅发布机制
3. DataStore提供了配置和状态数据的保存和查找机制

南向协议插件和北向应用的生命周期应该是一样的，每个南向插件都是一个OSGI的Bundle，遵循同样的设计思路，围绕MD-SAL架构提供的基础服务，通过YANG模型来定义其交互的接口标准。

