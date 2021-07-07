# RPC

## RPC框架主要工作流程
![rpc_主要概念](./imgs/rpc_主要概念.png)

**服务注册与发现**
- `服务生产者(Provider)`启动后，将机器与服务元信息注册到`注册中心(Registry)`
- `服务消费者(Consumer)`引入服务后，根据服务坐标在`注册中心(Registry)`查询服务提供者列表

**服务调用流程**
![rpc_调用流程](./imgs/rpc_调用流程.png)

- 负载均衡: 当服务有多个提供者时，`服务消费者(Consumer)`需要根据一定的负载均衡策略，选择一个`服务生产者(Provider)`进行远程调用
- 组装报文体实体: 
  - 请求报文实体一般包括: 请求的方法路径、参数列表(类型 + 值)、请求序列号等信息
- 序列化: 将请求报文实体序列化为二进制数据
- 发送请求: 通过Socket HTTP等方式将报文发送至负载均衡筛选出的服务提供者
- 反序列化请求报文: 将请求报文中的报文体，反序列化为请求报文对象
- 执行本地方法:
  - 通过请求报文对象的方法路径、参数列表，反射执行目标Method
- 组装响应报文实体:
  - 响应报文实体一般包括: 方法路径、返回值、是否异常、异常信息等
- 服务消费者接受响应后，反序列化为响应报文实体，返回反序列化后的方法返回值
----

## RPC通信协议的设计

**无论何种RPC框架，通信的协议格式大同小异，均需要有以下几点：**

![rpc_报文协议](./imgs/rpc_报文协议.png)

- 魔数(Magic): 约定这这是一个合法的协议类型，如果魔数不一致，直接拒绝连接
- 请求类型(type): 标示报文的类型。请求、响应、心跳等
- 报文体长度(body_length): 用来处理TCP粘包拆包的问题，需要让处理器知道，本次需要处理的报文体长度是多少
- 报文体(body): 实际传输的报文内容，基于一定方式序列化(serial)的二进制数据

----

## RPC中关键的技术点

- **Netty**
  - 基于Netty实现，服务端IO多路复用，实现单机高性能长连接
  - 内置TCP粘包拆包等解决方案
  - 替代方案: 基于HTTP、Socket BIO通信
- **序列化**
  - 将报文对象以统一的格式进行传输与解析
  - Java Seriallizable、Thrift、ProtoBuffer、JSON、Hessian等
- **动态代理**
  - 实现Consumer只依赖Provider的服务定义JAR包即可完成远程调用
  - 封装 序列化、负载、通信、失败处理 逻辑
  - 基于动态代理生成动态代理实现类
- **反射**
  - Provider根据请求报文中的方法坐标信息和参数类型信息，反射执行本地目标方法
- **负责均衡**
  - 当Provider集群部署的情况下，Consumer需要根据一定的负载均衡规则，筛选出一个目标服务提供者
  - 常见负载均衡策略:
    - 随机
    - 轮询
    - 加权轮询: 连接数、成功失败率、系统负载等条件
    - 一致性hash
- **心跳机制**
  - 维持服务端与客户端的可连通性方案，保证高可用
  - 不需要忙检测: 如果Consumer和Provider频繁进行通信，则无需进行心跳
  - 如何实现？
    - 为每一个连接设置两个标志位: 最后写时间 最后读时间
    - 基于TimerTask或Netty时间轮，定时执行任务，检查两个标志位的时间差，如果大于心跳间隔才发起检测
- **容错机制**
  - 保证服务调用的高可用的一种方案
  - 常见容错机制:
    - `failfast:` 快速失败，发起一次调用，失败即结束
    - `failover:` 失败自动切换，尝试其他服务器；retry的方式，一般适用于读服务
    - `failsafe:` 失败忽略，适用于日志写入；oneway的方式，只管发
    - `forking:` 多台机器并发调用，一台成功即成功
    - `broadcast:` 广播模式，逐个调用，一台失败全失败
  - `failfast` 和 `failover` 较为常见，`oneway`在不重要场景也可以使用
----

## RPC中注册中心的选型思考

### CP模型
**代表：Zookeeper**
- CP模型保证的是强一致性和分区容错性
- 特点：
  - 强一致性数据保证
  - zk的分布式一致性算法与raft相似，需要一半节点确认，而且会发生脑裂问题；
  - 集群中一半的节点不可用，会导致整个注册中心集群失效
- 更适合作为配置中心
  - zk可以作为配置中心 + 注册中心共存的方案
  - 注册中心强一致的必要性？
- Dubbo选用zk的原因？
  - 历史问题: Dubbo初始阶段没有很好地AP选项，zk的强一致性是可以作为注册中心方案
  - zk临时节点特性，可以让consumer快速感知provider的上下线状态

### AP模型
**代表：Eureka**
- AP模型是高可用性和分区容错性
- 特点：
  - 保证注册中心集群的高可用，各个节点互相独立，实现简单；
  - 并不会因为注册中心集群某一部分节点不可用，导致整个注册中心集群不可用
  - 基于定时同步机制，一定存在数据不一致情况；但是可以通过RPC的容错机制保证，例如retry、robbin等或者降级策略来保证服务调用间的高可用

----

## Dubbo框架

### Dubbo为何高性能
- 通信模型
  - 基于Netty实现NIO通信框架，IO多路复用；单机长连接+序列化，二进制传输
  - 单机长连接: consumer与每一个provider建立长连接
- 线程模型
  - IO线程与工作线程分离，异步化，future/callable模式
- 内存模型
  - 大量使用本地缓存，避免反射、动态代理带来的性能消耗
- 自定义的高效SPI框架，避免Java SPI带来的不灵活：每一个实现都会被初始化
- 高可用模型
  - 心跳机制: 双向心跳
  - 容错机制: 快速失败、自动切换、重试
  - 超时机制
  - 优雅下线机制

----

### Dubbo服务上下线策略(优雅停机、客户端感知后操作)

#### 引入Provider服务流程，以Spring容器为例
- **启动服务容器，连接zk，获取`/dubbo`下的service和provider的url信息**

- **`Invoker`是核心，Client对于每个Service的每一个Provider都会包装成为一个Invoker**

- **一个Service的多个Invoker被包装成一个`Cluster`**

- **`ReferenceBean.getObject()`基于动态代理生成代理对象，包含一个`Cluster`**

- **当执行远程方法调用时，实际上是通过`Cluster`完成与目标Provider的通信**
  - Cluster负责基于一定的负载均衡策略，选择一个Invoker进行通信
  - Cluster包含负载均衡 + 重试策略

- **当Provider上线或下线后，客户端zk监听器`NotifyListener`刷新**
  - 采用类似CopyOnWriteArrayList的读写分离策略，先创建新的invoker列表
  - 将invokers指向新的列表
  - 销毁旧的invoker

#### Provider下线流程，以Spring容器为例
**1. `ShutdownHookListener`监听Spring容器的ClosedEvent事件**

**2. `ZookeeperRegistry.destory()`将服务的注册信息在zk中移除并关闭与zk的连接**
  - dubbo注册的是临时节点，连接关闭，节点也被删除
  - 目的: 新的client端不再与当前的节点建立通信连接，以及Clinet端缓存新的Provider列表

**3. `DubboProtocol.destroy()`首先将当前的provider关闭，之后再关闭对下游调用的client**
  - 发送`sentReadOnlyEvent`: 通知consumer该连接已不可用，不要再发送新的请求(该过程是轮询发送且是oneway方式)
  - 指定一个超时时间，将provider的还在进行的线程执行完毕
  - 关闭自身Server，再关闭对下游的依赖: 否则上游此时再有请求进来，会导致失败

**4. Dubbo默认实现的问题:**
  - `DubboProtocol.destroy()`只有一个上限: 超时时间**或**此时没有工作线程，就会强制关闭。
    - `sentReadOnlyEvent`是一个oneway的过程，不知道client有没有收到 && 收到到摘除invoker也有时间差 
    - provider从注册中心摘除和consumer移除该provider的Invoker过程有时间差，此时可能consumer仍然有请求发送到已经关闭的provider上，
    - 而provider在这个时间差内，因为当前没有工作线程直接就结束，导致调用失败
  - 根据dubbo官网描述，在容器中需要调用`DubboShutdownHook`的逻辑
  - 解决方案: 
    - 在`DubboProtocol.destroy()`过程前，设置一个停机等待的下限: 因为第一步已经断开了zk的连接，会通知consumer移除该provider的invoker，提供一个缓冲时间
    - 自定义监听Spring容器的ClosedEvent事件，手工新增ShutdownHook
    - Kill -9 无解，也需要发布平台一起配合。如果对Hook改造了停机下限，那么发布平台强制停止的时间不能小于这个值
----

### Dubbo与Spring Cloud
- RPC功能都是Dubbo和Spring Cloud中的一个子集
  - RPC只关心进程间通信
  - Dubbo/Spring Cloud: 微服务治理的全套解决方案
- Dubbo更注重语言级别的远程方法调用，并没有像Spring Cloud提供网关、限流、Eureka的解决方案；但提供了较为成熟的Dubbo Admin
- 在Spring cloud体系中，也可以融合Dubbo；Dubbo + zk/Nacos代替fegin + eureka + robbin
- 没有谁优谁劣，按需使用，考虑Team的技术演进、技术与运维能力
- Spring Cloud的通讯基于Http协议、Dubbo自定义协议 + NIO长连接

----
## 关于ServiceMesh的一些思考
- 在很多rpc框架中，通信、负载、容错、序列化等功能全部与应用耦合在一起，当rpc框架升级时，应用服务必然受到影响，会降低服务SLA
- ServiceMesh的理念，个人认为，在Consumer和Provider建立起一个Proxy
  - Proxy直接完成进程间通讯、负载、容错 - 即 SideCar
  - 应用端只需要维护与Proxy的通讯、序列化流程，这种核心流程一旦稳定近乎不会改变
  - 伴随着中间件的升级，只需要更新Proxy即可；建立公共Proxy，保证服务SLA不受影响

----
## SOA与微服务
- **SOA与微服务的目标均在于应用服务的解耦与服务化的解决方案**
- **SOA与微服务的主要区别**
  - 在SOA体系中，服务间通讯主要依赖ESB企业服务总线。而ESB是作为一个单点组件存在的，且限制了技术栈必须为Java或某一周ESB支持的技术；
  - 微服务体系中，服务网关类似于ESB的功能；但是服务网关可以通过HTTP这种无绑定技术的协议进行通讯；同时提供限流、安全、版本管理等功能
  - SOA更面相于上层企业架构的解耦；而微服务的粒度更新，在同一个领域中，可以有N多的微服务组成，更易于持续集成、架构松耦合、高内聚

----

**参考：**
> Dubbo——服务调用、服务暴露、服务引用过程: https://www.jianshu.com/p/15e77db72b75
>
> 优雅停机: https://dubbo.apache.org/zh/docs/v2.7/user/examples/graceful-shutdown/