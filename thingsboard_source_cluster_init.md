### 前言

本篇主要对ThingsBoard源码的目录结构做一个大致介绍，其中会涉及到服务的划分。

**注：笔者所编译的版本为github 2021/7/7拉取的版本，ThingsBoard版本=3.3.0，操作系统 Mac OS X**

在开始本篇的内容之前，需要阐述ThingsBoard中的一些关键概念：

### 关键词

- **partition**  

  属于ThingsBoard抽象的消息队列定义中的概念之一。

  partition除了用来提升消息并行处理能力之外，在ThingsBoard中尤为重要的功能就是支持让entity id的hash值与分区数进行取模后确定实体对应的分区，这样就能实现同一个实体的消息始终发送到同一个分区的效果。同时ThingsBoard自研的消费者协调逻辑通过zookeeper和一致性hash算法确定集群中的某个节点应该订阅那些分区的数据。

- **topic**  

  属于ThingsBoard抽象的消息队列定义中的概念之一。

  topic可以理解为rest接口中的url，也可以对等理解为kafka中的topic

- **entity**  
  平台根据各种业务操作定义出的实体，包含：

  - TENANT（租户）   
  - CUSTOMER（客户）  
  - USER（用户）  
  - DASHBOARD（仪表盘）  
  - ASSET（资产）  
  - DEVICE（设备)   
  - ALARM（告警）   
  - RULE_CHAIN（规则链）   
  - RULE_NODE（规则节点）   
  - ENTITY_VIEW（实体视图）   
  - WIDGETS_BUNDLE（视图工具包）   
  - WIDGET_TYPE（视图工具类型）

- **entityId**  
  上述实体的id，服务间消息应该发送到哪个分区，就是通过将entityId哈希后与分区数取余所得

- **serviceId**  
  为每个服务设置的id，如果没有明确指定，则取当前服务的`hostname`

- **ServiceType**  
  平台定义的服务类型  

  - TB_CORE（核心服务）
  - TB_RULE_ENGINE（规则引擎）  
  - TB_TRANSPORT（传输服务MQTT/HTTP/COAP）  
  - JS_EXECUTOR（JS执行器(用户自定义脚本)）

- **isolatedTenant**  
  指定该服务只处理某一个租户的业务

- **QueueInfo**  

  描述具体的`serviceType`所要订阅的topic和分区

  serviceType - messageName - topic - partition  
  服务间消息的配置，`name`，`topic`，`partition`

- **ServiceInfo**  
  服务的基本信息，包含了`serviceId`、`isolatedTenant`、`ServiceType`，如果当前`ServiceType`是`RULE_ENGINE`，还会包含规则引擎服务定义的`QueueInfo`集合

- **ServiceQueue**
  
  属于ThingsBoard抽象的消息队列定义中的概念之一。
  
  每种服务用于通讯的队列定义，包括`ServiceType`、`messageName`(消息的名称)，一种服务可能针对不同的功能会有多个消息。
  
- **ServiceQueueKey**  
  将ServiceQueue与tenantId进行绑定，可对指定租户提供服务

- **partitionTopics**  
  缓存ServiceQueue与topic，即本节点所提供的服务与各个消息(`messageName`)对应的topic

- **partitionSizes**  
  缓存ServiceQueue与partition，即本节点所提供的服务与各个消息(`messageName`)的分区数

- **myPatitions**  
  通过计算后，缓存的本节点需要提供的服务以及服务下各个消息(`messageName`)需要发布/订阅的分区

- **currentPartitions**
  当前节点负责的topic及分区`Set`集合

- **currentOtherServices**
  缓存的除当前节点以外的其他节点`List<ServiceInfo>`

### 集群生命周期

**Thingboard**的集群管理是通过zookeeper来实现的，通过启动的时候往zookeeper的`/thingsboard/nodes/${serviceInfo.toByteArray()}`中创建`EPHEMERAL_SEQUENTIAL`临时序列节点，同时`watch`这个目录的值变化，以实现监听集群节点上下线事件，并做对应处理。

#### **服务实例上线流程**

1. `DefaultTbServiceInfoProvider`根据配置初始化`ServiceInfo`，主要是节点的服务类型和租户隔离时的租户id

2. `HashPartitionService`根据配置，初始化核心服务与`rule-engine`服务的topic名称和分区数量配置，并分别放入`partitionTopics`和`partitionSizes`中

3. `ZKDiscoveryService`通过`@PostConstruct`根据配置中的zk.*配置项启动zookeeper客户端，并监听`/thingsboard/nodes`目录节点变化情况

4. `ZKDiscoveryService`监听springboot应用启动完成事件`ApplicationReadyEvent`，此段逻辑主要执行两个方法：

   ```
   @EventListener(ApplicationReadyEvent.class)
       @Order(value = 1)
       public void onApplicationEvent(ApplicationReadyEvent event) {
           //。。。删除了无用的代码
           //注册服务
           publishCurrentServer();
           log.info("Going to recalculate partitions...");
           //集群节点发生变更时重新计算分区
           recalculatePartitions();
       }
   ```

   1. `publishCurrentServer()`

      1. 注册当前服务至zookeeper的`/thingsboard/nodes/${serviceInfo.toByteArray()}`，并监听zk客户端连接状态自动重连

   2. `recalculatePartitions()`

      1. 调用`HashPartitionService.recalculatePartitions(currentService,otherServices)`重新计算服务节点与分区

      2. `otherServices`即为`/thingsboard/nodes/`路径中与上一步创建的路径不同的节点

      3. 调用`addNode(Map<ServiceQueueKey, List<ServiceInfo>> queueServiceList, ServiceInfo instance)`更新本地缓存的服务与队列名称到`queueServicesMap`中，这个map保存的是某个租户下某种类型服务的实例列表，并对rule-engine类型的服务做特殊处理，因为此种类型的服务会有多个队列。

      4. 将`queueServicesMap`缓存的实例列表以serviceId排序

      5. 将`myPartitions`转存到`oldPartitions`并重新初始化`myPartitions`，接下来将要计算本实例应该订阅的topic与对应的分区

      6. `partitionSize`取出配置的ServiceQueue与其分区数并遍历，ServiceQueue参加上方的关键词解释。

         ```java
         partitionSizes.forEach((serviceQueue, size) -> {
         		//根据TenantId租户id组合ServiceQueue生成`ServiceQueueKey`，用于实现租户隔离
         		ServiceQueueKey myServiceQueueKey = new ServiceQueueKey(serviceQueue, 			myIsolatedOrSystemTenantId);
           	//循环分区
         		for (int i = 0; i < size; i++) {
                 //根据`ServiceQueueKey`从4.2.3步骤缓存中获取服务节点列表。
                 //用当前分区和服务实例数取余，计算当前分区应该被哪个服务实例订阅
         				ServiceInfo serviceInfo = resolveByPartitionIdx(queueServicesMap.get(myServiceQueueKey), i);
         				//如果计算出的服务实例是当前服务实例，就缓存该服务实例与分区的关系到myPartitions中
                 if (currentService.equals(serviceInfo)) {
         						ServiceQueueKey serviceQueueKey = new ServiceQueueKey(serviceQueue, getSystemOrIsolatedTenantId(serviceInfo));
         						myPartitions.computeIfAbsent(serviceQueueKey, key -> new ArrayList<>()).add(i);
         				}
         		}
         });
         ```

      7. 遍历`oldPartitions`与`myPartitions`进行比较，如果发现当前实例所属的服务类型有变更（如从core-serviice变成transport+rule-engine），或者负责的分区有变更，就通过spring发布一个`PartitionChangeEvent`  

      8. 将收到的 `ohterServices` 与 缓存的 `currentOtherServices` 比较看是否有变更的节点，如果有的话，代表集群中其服务实例也产生了变化，也就是集群的拓扑结构产生了变化，通过spring发送`ClusterTopologyChangeEvent`事件

         - `DefaultTbLocalSubscriptionService`，重新处理websocket会话与分区/设备的关系

      9. 发送`ServiceListChangedEvent`事件，下一章节着重介绍7、8、9发送的三个事件。

5. 至此，服务就加入了集群中。具体各个服务启动时依赖的其他组件，如Kafka的初始化，详见各个项目的README

#### 实例上线产生的事件

##### PartitionChangeEvent

- `DefaultActorService`销毁针对旧分区的actor，为新分区和topic创建新的actor  
- `DefaultSubscriptionManagerService`、`DefaultTelemetrySubscriptionService`、`DefaultTbLocalSubscriptionService`重新缓存 `currentPartitions`，移除每个分区下的websocket订阅
- `DefaultDeviceStateService`重新缓存设备状态，根据设备id哈希后计算所属分区，仅缓存应属于当前分区下的设备
- `DefaultTbCoreConsumerService`、`DefaultTbRuleEngineConsumerService`在消息队列中重新订阅 topic=topic.tenantId.partition

##### ClusterTopologyChangeEvent

##### ServiceListChangedEvent

 **服务下线流程**

 1. `ZKDiscoveryService`通过@PreDestroy监听springboot的生命周期
   2. 调用destroyZkClient()在zookeeper上删除自身节点
   3. 其他服务监听zookeeper节点的变化，重复上线流程***4.iii***流程以重新计算分区、分配actor、更新设备状态、更新订阅等
      ###用户登陆流程  
      用户输入账号密码--HttpRequest--FilterChainProxy--Filter--AuthenticationToken(未认证)--AuthenticationManager--Provider--UserDetailsService--UserDetails--AuthenticationToken(已认证)-
 4. 用户登陆校验过程使用spring-security实现
 5. 平台默认使用用户名密码验证后通过jwt通讯
 6. 平台自定义的filter负责拦截不同的请求，校验请求方式与参数是否合法，并转化为对应的参数对象`UsernamePasswordAuthenticationToken`,`JwtAuthenticationToken`,`RefreshAuthenticationToken`
 7. 平台自定义的`provider`根据其自身的`support()`方法来判断处理什么样的校验请求

   - `RestAuthenticationProvider`处理`UsernamePasswordAuthenticationToken`类型的校验逻辑
     1. 根据邮箱查询User
     2. 根据UserId查询UserCredentials
     3. 根据UserCredentials中的密码与用户输入的密码进行匹配
     4. 如果密码不正确，查询SecuritySettings配置看是否超出错误次数发邮件告警
     5. 校验通过更新UserCredentials中用户token，更新最后登陆时间
     6. 判断密码是否过期
     7. SecurityUser对象到UsernamePasswordAuthenticationToken
   - `JwtAuthenticationProvider`处理`JwtAuthenticationToken`类型的校验逻辑
     - 解密token，拿到对象放入JwtAuthenticationToken
   - `RefreshTokenAuthenticationProvider`处理`RefreshAuthenticationToken`类型的校验逻辑
     - 校验refreshToken，通过后重新下发token
       ###数据在集群内处理流程
       **设备连接流程-MQTT**

 1. 设备发送CONNECT报文至`transport-mqtt-api`
 2. `transport-mqtt-api`通过netty提供MQTT-Broker服务，其中mqtt协议由`MqttDecoder`，`MqttEncoder`实现，`MqttTransportHandler`负责处理所有收到的消息
 3. MqttTransportHandler将解析到的accessToken放入ValidateDeviceTokenRequestMsg通过`DefaultTbQueueRequestTemplate.send(),`使用kafka发送到`tb_transport.api.requests`，并将请求放入`pendingRequests`
    消息中包含了发送此消息的节点id(nodeId)，以保证core服务发送响应的时候，是该transport节点收到响应
 4. core服务的TbCoreTransportApiService监听spring的ApplicationReadyEvent
 5. TbCoreTransportApiService通过DefaultTbQueueResponseTemplate启动线程从`tb_transport.api.requests`topic拉取数据，将数据交由`DefaultTransportApiService.hanele()`处理
 6. `DefaultTransportApiService.hanele()`匹配到`transportApiRequestMsg.hasValidateTokenRequestMsg()`,通过AccessToken查询出Device信息后，根据请求中的nodeId，包装成`TransportApiResponseMsg`通过kafka发送到`tb_transport.api.responses.$nodeId`
 7. transport服务的`DefaultTbQueueRequestTemplate`会启动线程拉取`tb_transport.api.responses.$nodeId`的数据，根据数据中的requestId从`pendingRequests`中获取发送时放入的异步响应对象，将数据放入其中，触发其onSuccess或onFailure事件
 8. 校验通过，调用`MqttTransportHanlder.onValidateDeviceResponse()`将设备和租户相关信息缓存到本地
 9. `checkGatewaySession()`检查当前设备是不是智能网关,如果是智能网关,新建一个处理网关子设备请求得handler-`GatewaySessionHandler`   