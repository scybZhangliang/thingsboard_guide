本篇主要对ThingsBoard源码的目录结构做一个大致介绍，其中会涉及到服务的划分。

**注：笔者所编译的版本为github 2021/7/7拉取的版本，ThingsBoard版本>=3.3.0，操作系统 Mac OS X**

### 一、目录结构

```
.		根目录,定义了根pom,声明公共的库,插件,依赖等   
├── application 			主要业务逻辑的实现，如实体的增删改查与对外提供的接口
├── common						公共的一些类,工具,数据库接入实现,连接层实现等  
│   ├── actor						ThingsBoard自己实现的actor模式，基于线程池实现（3.2版本前使用的是akka提供的实现方案）
│   ├── cache						缓存实现，根据不同的配置提供不同的缓存 caffeine/redis-cluster/redis-standalone
│   ├── coap-server			COAP协议基本实现（端口的监听、URI校验等）
│   ├── dao-api					定义了数据操作的公共接口  
│   ├── data						公共的实体类  VO/DTO  
│   ├── edge-api				用于模拟边缘侧向平台发送grpc消息的客户端
│   ├── message					服务间通讯以及服务注册发现等需要用到的实体类/工具类,服务间传输的类定义在src/proto中  
│   ├── queue						服务间消息传输方式的实现,包括kafka/memory/pubsub/rabbitmq等,通讯的消息载体定义在src/proto中 
│   ├── stats						平台统计功能使用的一些类与配置
│   ├── transport			设备数据传输服务的实现,这里是公共依赖  
│   │   ├── coap				coap协议在tb内的实现，即tb内定义的coap实体、连接鉴权、url匹配处理等
│   │   ├── http				http协议在tb内的实现
│   │   ├── lwm2m			  lwm2m协议在tb内的实现
│   │   ├── mqtt				mqtt协议在tb内的实现
│   │   ├── snmp				snmp协议在tb内的实现
│   │   └── transport-api	传输服务的公共接口定义与配置，以及传输服务公用的一些实现，如鉴权、限流等
│   └── util·						公共工具  
├── dao								数据库接入的真正实现  
├── docker						平台在docker-compose环境部署需要的脚本与配置
│   ├── haproxy					在docker中使用haproxy负载服务
│   │   └── config
│   ├── monitoring			在docker中使用grafana与Prometheus监控服务
│   │   ├── grafana
│   │   └── prometheus
│   ├── tb-node					平台ui+后台服务
│   │   └── conf
│   └── tb-transports		平台各传输服务
│       ├── coap
│       ├── http
│       ├── lwm2m
│       ├── mqtt
│       └── snmp
├── img								静态资源
├── k8s								在kubernetes环境部署需要的脚本与配置
│   ├── basic						使用deployment方式部署kafka、zk、redis，数据不会被持久化，非高可用
│   ├── common					数据库+平台+传输服务
│   └── high-availability  使用statefulset+headless service方式部署kafka、zk、redis，数据会被持久化，高可用
├── msa								以docker将平台拆分出的微服务容器化
│   ├── black-box-tests		测试容器部署
│   ├── js-executor				平台规则引擎中脚本动态脚本的执行服务，使用nodejs实现
│   ├── tb								docker部署平台的脚本与配置
│   ├── tb-node						平台容器化的dockerfile
│   ├── transport					各传输服务容器化的dockerfile
│   │   ├── coap
│   │   ├── http
│   │   ├── lwm2m
│   │   ├── mqtt
│   │   └── snmp
│   └── web-ui						平台ui容器化的dockerfile
├── netty-mqtt				java实现的mqtt客户端，用于模设备连接以及性能测试
├── packaging					打包脚本，主要是用于安装过程中打Windows包与linux包
│   ├── java
│   │   ├── assembly
│   │   ├── filters
│   │   └── scripts
│   └── js
│       ├── assembly
│       ├── filters
│       └── scripts
├── rest-client				java实现的http-client，用于模拟设备连接以及性能测试
├── rule-engine				规则引擎实现，
│   ├── rule-engine-api					公共接口与公共配置
│   └── rule-engine-components	各规则节点的实现，如kafka转发、动态脚本执行、邮件发送等
├── tools							一些公共工具，shell脚本、Python脚本等
├── transport					使用springboot运行各传输服务
│   ├── coap
│   ├── http
│   ├── lwm2m
│   ├── mqtt
│   └── snmp
└── ui-ngx						平台的UI
```

### 二、平台架构

首先贴一下官网的架构介绍。https://thingsboard.io/docs/reference/

**Thingsboard**支持集中式部署与微服务部署两个方式，集中式部署即整个平台部署为一个单体项目，所有协议实现、业务处理、规则引擎等都集中在一个可执行程序中。而微服务部署则将规则引擎、不同的协议实现、业务处理拆分成不同的微服务，各个微服务通过消息队列以protobuf将数据序列化后进行通讯，至于为什么要用消息队列而不是RPC进行服务间通讯，可以参见官网的解释：

```
Note: Starting version 2.5 we have switched from using gRPC to Message Queues for all communication between ThingsBoard components. The main idea was to sacrifice small performance/latency penalties in favor of persistent and reliable message delivery and automatic load balancing.
```

大致意思就是牺牲一定的延迟换取消息队列天然支持的数据的持久化与可靠性保证，还有自动的负载均衡。

------

#### ThingsBoard自身架构

可以看到无论是单体架构还是微服务架构，ThingsBoard内部定义的处理单元都分为tb-node、rule-engine、transport，每个服务间使用使用kafka进行数据交互，并使用Protobuffer对数据进行序列化与反序列化。接下来介绍每个具体的服务

- tb-node
  - 对上层应用和ui服务提供rest接口
  - 处理websocket订阅，主动推送收到的设备遥测和设备属性变化
  - 调用规则引擎处理收到的消息，如RPC、设备遥测、设备上下线等
  - 维护设备在线状态

- rule-engiien
  - 根据配置的规则链处理所有的[消息](https://thingsboard.io/docs/user-guide/rule-engine-2-0/overview/#rule-engine-message)
- 传输服务
  - mqtt协议实现
  - http协议实现
  - coap协议实现
  - 。。。
- js-executor
  - 通过rest接口与kafka订阅的方式获取脚本动态执行并返回结果
- web-ui
  - 设备、租户、客户、资产等的管理
  - 通过规则引擎配置设备相关的数据流
  - 通过仪表板与组件库组合系统中的数据
  - 通过websocket订阅动态更新的数据

在ThingsBoard的设计中，每种类型的服务实例，在集群中都是对等的。各个服务实例在zookeeper中注册后，通过一致性hash算法结合约定的topic+分区机制，实现集群的水平扩展，具体的集群初始化流程，我会在后面的章节中说明。

#### 依赖的第三方组件

##### 时序数据库

ThingsBoard提供三种时序数据存储方案，用于存储设备遥测、属性、日志等需要关注时间变化的数据。可根据实际情况进行配置。

- postgresql

  依赖postgresql本身的表分区功能，按时间对表切片

- timescaledb

  postgresql的一个扩展，安装后可以像使用普通表一样使用数据数据库

- Cassandra

  需要单独安装的专业时序数据库

##### kafka

服务间的数据流转，用于替代GRPC。牺牲一定的延迟以换取消息的持久化与稳定性，和kafka本身的水平扩展、高可用的特性。

##### redis

缓存资产、视图、设备、设备证书、会话信息、设备关系等

##### zookeeper

1. kafka集群需要
2. 用作ThingsBoard集群的服务发现与注册中心

##### HAProxy

软负载