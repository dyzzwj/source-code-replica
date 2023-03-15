																						nacos注册中心服务端

# 前言

本章分析Nacos1.4.1注册中心服务端，**主要关注临时实例（Instance.ephemeral=true，默认情况）**

- 服务端模型：ServiceManager、Service、Cluster、Instance
- Distro协议：服务于注册中心的AP协议
- 服务查询：UDP监听、保护模式
- Nacos集群管理：集群初始化，集群健康检查
- Distro协议对于写请求的处理方式
- 服务注册：节点本地数据更新、集群数据同步、UDP推送客户端
- 客户端心跳：心跳超时处理
- 集群数据同步：Nacos如何保证每个节点数据一致

# 一、服务端模型

从逻辑上，命名服务的模型如下。

![命名服务逻辑模型.png](../images/07157b5e770040a2ac4b4e8313f23181~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

服务端与客户端的主要不同点在于，客户端的NamingService是针对某个Naocs集群的某个Namespace创建的，客户端使用过程Namespace是不变的。从服务端实现的角度看，如下。

![命名服务实现模型.png](../images/f42b79d1f8ae4a9686fdc35669882ace~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**ServiceManager单例，管理Namespace-Group-Service的映射关系**，其中Map<String, Service>的Key是groupName@@serviceName。

```java
@Component
public class ServiceManager implements RecordListener<Service> {
    // namespace - groupName@@serviceName - Service
    private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
}

```

**com.alibaba.nacos.naming.core.Service存储了Namespace和Group和Service的所有信息**。

```java
// com.alibaba.nacos.naming.core.Service
public class Service extends com.alibaba.nacos.api.naming.pojo.Service implements Record, RecordListener<Instances> {
    // 所属namespace
    private String namespaceId;
}
// com.alibaba.nacos.api.naming.pojo.Service
public class Service implements Serializable {
    // groupName@@serviceName
    private String name;
    // 服务保护阈值，当大多数服务下线，认为当前注册中心节点发生故障，返回所有实例，包括非健康实例
    private float protectThreshold = 0.0F;
    // 分组
    private String groupName;
}
```

此外**com.alibaba.nacos.naming.core.Service管理其下的所有Cluster**。

```
// Cluster注册表，key是集群名称
private Map<String, Cluster> clusterMap = new HashMap<>();
```

**Cluster管理所有持久实例和临时实例**。

```java
// com.alibaba.nacos.naming.core.Cluster
public class Cluster extends com.alibaba.nacos.api.naming.pojo.Cluster implements Cloneable {
    // 持久Instance
    private Set<Instance> persistentInstances = new HashSet<>();
    // 临时Instance
    private Set<Instance> ephemeralInstances = new HashSet<>();
    // 所属Service
    private Service service;
}
// com.alibaba.nacos.api.naming.pojo.Cluster
public class Cluster implements Serializable {
    /**
     * Name of belonging service.
     */
    private String serviceName;
    /**
     * Name of cluster.
     */
    private String name;
}
```

**Instance服务实例，客户端服务注册的单位**。

```java
// com.alibaba.nacos.naming.core.Instance
public class Instance extends com.alibaba.nacos.api.naming.pojo.Instance implements Comparable {
    // 上次心跳时间
    private volatile long lastBeat = System.currentTimeMillis();
    // namespace
    private String tenant;
}
// com.alibaba.nacos.api.naming.pojo.Instance
public class Instance implements Serializable {
    // 唯一标识 = 分组+集群+服务+ip+端口
    private String instanceId;
    private String ip;
    private int port;
    private double weight = 1.0D;
    private boolean healthy = true;
    private boolean enabled = true;
    // 是否临时节点
    private boolean ephemeral = true;
    // 所属集群
    private String clusterName;
    // 服务名 = 分组+服务
    private String serviceName;
    // 元数据
    private Map<String, String> metadata = new HashMap<String, String>();
}
```



# 二、distro协议

Nacos注册中心采用distro协议，用于注册中心，AP。

**对于客户端，服务端每个节点是对等的，无论读写请求发往哪个节点都可以被处理**。如果某个节点处理失败，客户端会重新选择一个节点请求。客户端的请求主要包括：服务注册、服务查询、服务监听、心跳请求等。

```java
// NamingProxy#reqApi
public String reqApi(String api, Map<String, String> params, Map<String, String> body, List<String> servers,
        String method) throws NacosException {
    // 随机选择某个节点作为第一次请求节点
    /**
      * 对于客户端，服务端每个节点是对等的，无论读写请求发往哪个节点都可以被处理。如果某个节点处理失败，客户端会重新选择一个节点请求。
      * 客户端的请求主要包括：服务注册、服务查询、服务监听、心跳请求等。
      */
    // 随机选择某个节点作为第一次请求节点
    Random random = new Random(System.currentTimeMillis());
    int index = random.nextInt(servers.size());
    for (int i = 0; i < servers.size(); i++) {
      String server = servers.get(index);
      try {
        return callServer(api, params, body, server, method);
      } catch (NacosException e) {
        exception = e;
      }
      // 发生异常，选择下一个节点尝试请求
      index = (index + 1) % servers.size();
    }
   // ...
}
```

对于服务端要处理以下操作：

- **读**：集群**每个节点都存储所有数据**，**每个节点都可以处理读请求**，返回当前节点注册表里的数据，无论数据是否是最新的。（**GET /nacos/v1/ns/instance/list**）
- **写**：**以服务为单位，每个节点负责一部分服务**。从服务维度来说，如果当前节点负责这个服务，那么这个节点称为**责任节点**。客户端向随机服务端发起写请求后，服务端判断自己是否是责任节点，如果是的话自己处理，否则转发至其他节点。（**DistroFilter**）
- **客户端心跳：**服务端如果长时间未收到客户端心跳，则下线该服务实例（**ClientBeatCheckTask、ClientBeatProcessor**）；服务端如果收到客户端心跳，但服务不存在，执行注册逻辑（**BeatReactor、PUT /nacos/v1/ns/instance/beat**）。备注：**临时实例注册表是通过客户端主动发送心跳来维护的；持久实例是通过服务端主动做健康检查来维护的，这里只考虑临时实例注册**。
- **集群管理：**每个服务端节点主动发送健康检查到其他节点，响应成功的节点被该节点视为健康节点；另外健康检查也负责同步集群成员（**MemberInfoReportTask**）。
- **数据同步**：以服务为单位，责任节点处理完成写请求后，异步将数据同步给其他节点，保证最终所有节点的注册表信息一致（**DistroProtocol**）。

# 三、集群管理

集群管理是Nacos的通用功能，非注册中心独有。

**ServerMemberManager**负责Nacos集群管理。

```java
@Component(value = "serverMemberManager")
public class ServerMemberManager implements ApplicationListener<WebServerInitializedEvent> {
    // 所有nacos节点
    private volatile ConcurrentSkipListMap<String, Member> serverList;
    // nacos自己如何发现nacos服务
    private MemberLookup lookup;
    // 当前nacos节点
    private volatile Member self;
    // 健康状态的节点地址集合
    private volatile Set<String> memberAddressInfos = new ConcurrentHashSet<>();
    // 集群成员信息广播任务
    private final MemberInfoReportTask infoReportTask = new MemberInfoReportTask();
    private volatile boolean isInIpList = true;
    private int port;
    private String localAddress;
}
```

**Member**代表Nacos集群节点。

```java
public class Member implements Comparable<Member>, Cloneable {
    // ip
    private String ip;
    // port
    private int port = -1;
    // 状态
    private volatile NodeState state = NodeState.UP;
    // 扩展信息
    private Map<String, Object> extendInfo = Collections.synchronizedMap(new TreeMap<>());
    // ip:port
    private String address = "";
    // 健康检查连续失败次数
    private transient int failAccessCnt = 0;
}
```

NodeState，节点状态。

```java
public enum NodeState {
    STARTING,
    UP,
    SUSPICIOUS,
    DOWN,
    ISOLATION,
}
```

只有UP、SUSPICIOUS、DOWN三种状态在使用。

- UP：健康检查通过。
- SUSPICIOUS：健康检查失败，且失败次数小于一定阈值。
- DOWN：健康检查失败，且失败次数大于一定阈值。

## 1、集群初始化

一个新加入的节点，需要通过**MemberLookup**初始化内存中的集群列表ServerMemberManager.serverList，才能进行后续心跳发送加入集群。



```java
// ServerMemberManager
protected void init() throws NacosException {
    this.port = EnvUtil.getProperty("server.port", Integer.class, 8848);
    this.localAddress = InetUtils.getSelfIP() + ":" + port;
    this.self = MemberUtil.singleParse(this.localAddress);
    this.self.setExtendVal(MemberMetaDataConstants.VERSION, VersionUtils.version);
    serverList.put(self.getAddress(), self);
    // 注册集群变更通知监听器
    registerClusterEvent();
    // 初始化集群列表
    initAndStartLookup();
}
```

**MemberLookup**实现类有三个：

- **StandaloneMemberLookup**：当节点以standalone形式启动，直接取自身作为集群列表。
- **FileConfigMemberLookup**：取nacos.home/conf/cluster.conf配置文件中的内容作为集群列表。
- **AddressServerMemberLookup**：使用外部地址服务，为nacos集群提供服务发现能力，初始化集群列表。请求http://{address.server.domain}:{address.server.port}/{address_server_url}获取cluster.conf。

ServerMemberManager会根据当前情况选择合适的MemberLookup实现，执行start方法。

```java
// ServerMemberManager
private void initAndStartLookup() throws NacosException {
    //根据配置决定lookup实现类
    this.lookup = LookupFactory.createLookUp(this);
    this.lookup.start();
}
```

以**FileConfigMemberLookup**为例，start方法读取{nacos.home}/conf/cluster.conf，通过memberManager.memberChange(members)回调MemberManager。此外利用JDK的WatchService实现文件监听，当cluster.conf发生变化时会重新加载集群列表回调MemberManager。

```java
// FileConfigMemberLookup
@Override
public void start() throws NacosException {
    if (start.compareAndSet(false, true)) {
        readClusterConfFromDisk();
        try {
            //注册配置文件改变的监听  使用JDK的WatchService实现文件监听
            WatchFileCenter.registerWatcher(EnvUtil.getConfPath(), watcher);
        } catch (Throwable e) {
            Loggers.CLUSTER.error("An exception occurred in the launch file monitor : {}", e.getMessage());
        }
    }
}
private void readClusterConfFromDisk() {
    // 1. 读取{nacos.home}/conf/cluster.conf，加载tmpMembers
    Collection<Member> tmpMembers = new ArrayList<>();
    try {
        List<String> tmp = EnvUtil.readClusterConf();
        tmpMembers = MemberUtil.readServerConf(tmp);
    } catch (Throwable e) {
        Loggers.CLUSTER
                .error("nacos-XXXX [serverlist] failed to get serverlist from disk!, error : {}", e.getMessage());
    }
    // 2. 更新内存中的nacos节点列表，发布MembersChangeEvent事件。
    // 2. this.memberManager.memberChange(members)
    afterLookup(tmpMembers);
}
```

ServerMemberManager的**memberChange**方法更新内存中的nacos节点列表，发布**MembersChangeEvent**事件。

```java
// 健康状态的节点地址集合
private volatile Set<String> memberAddressInfos = new ConcurrentHashSet<>();
// 所有nacos节点
private volatile ConcurrentSkipListMap<String, Member> serverList;
synchronized boolean memberChange(Collection<Member> members) {
    if (members == null || members.isEmpty()) {
        return false;
    }

    boolean isContainSelfIp = members.stream()
        .anyMatch(ipPortTmp -> Objects.equals(localAddress, ipPortTmp.getAddress()));

    if (isContainSelfIp) {
        isInIpList = true;
    } else {
        isInIpList = false;
        members.add(this.self);
        Loggers.CLUSTER.warn("[serverlist] self ip {} not in serverlist {}", self, members);
    }

    // If the number of old and new clusters is different, the cluster information
    // must have changed; if the number of clusters is the same, then compare whether
    // there is a difference; if there is a difference, then the cluster node changes
    // are involved and all recipients need to be notified of the node change event

    boolean hasChange = members.size() != serverList.size();
    ConcurrentSkipListMap<String, Member> tmpMap = new ConcurrentSkipListMap<>();
    Set<String> tmpAddressInfo = new ConcurrentHashSet<>();
    for (Member member : members) {
        final String address = member.getAddress();

        if (!serverList.containsKey(address)) {
            hasChange = true;
        }

        // Ensure that the node is created only once
        tmpMap.put(address, member);
        if (NodeState.UP.equals(member.getState())) {
            tmpAddressInfo.add(address);
        }
    }
    //更新内存中的nacos节点列表
    serverList = tmpMap;
    memberAddressInfos = tmpAddressInfo;

    Collection<Member> finalMembers = allMembers();

    Loggers.CLUSTER.warn("[serverlist] updated to : {}", finalMembers);

    // Persist the current cluster node information to cluster.conf
    // <important> need to put the event publication into a synchronized block to ensure
    // that the event publication is sequential
    if (hasChange) {
        MemberUtil.syncToFile(finalMembers);
        Event event = MembersChangeEvent.builder().members(finalMembers).build();
        //发布MembersChangeEvent事件。
        NotifyCenter.publishEvent(event);
    }

    return hasChange;
}
```



## 2、集群健康检查



ServerMemberManager实现了ApplicationListener接口，关注**WebServerInitializedEvent**事件。

![ServerMemberManager监听Spring事件.png](../images/fa84295a14ad411fb4687e74c157716c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**Tomcat启动后**会回调ServerMemberManager，开启一个**MemberInfoReportTask**当前节点信息广播任务。

```java
// ServerMemberManager
// 集群成员信息广播任务
private final MemberInfoReportTask infoReportTask = new MemberInfoReportTask();
public void onApplicationEvent(WebServerInitializedEvent event) {
    getSelf().setState(NodeState.UP);
    if (!EnvUtil.getStandaloneMode()) {
        GlobalExecutor.scheduleByCommon(this.infoReportTask, 5_000L);
    }
    EnvUtil.setPort(event.getWebServer().getPort());
    EnvUtil.setLocalAddress(this.localAddress);
}
```

MemberInfoReportTask，**每2s执行POST /v1/core/cluster/report**，向集群随机节点（包含DOWN）发送当前节点信息（Member），一方面是为了**同步当前节点信息**，另一方面也是**健康检查**。

```java
class MemberInfoReportTask extends Task {
    private final GenericType<RestResult<String>> reference = new GenericType<RestResult<String>>() {
    };
    private int cursor = 0;
    @Override
    protected void executeBody() {
        // 获取除当前节点以外其他所有节点，包含down的
        List<Member> members = ServerMemberManager.this.allMembersWithoutSelf();
        if (members.isEmpty()) {
            return;
        }
        // 轮询选择
        this.cursor = (this.cursor + 1) % members.size();
        Member target = members.get(cursor);
        // 调用/v1/core/cluster/report，将getSelf自己的Member信息传给对端
        final String url = HttpUtils
                .buildUrl(false, target.getAddress(), EnvUtil.getContextPath(), Commons.NACOS_CORE_CONTEXT, "/cluster/report");

        try {
            asyncRestTemplate
                    .post(url, Header.newInstance().addParam(Constants.NACOS_SERVER_HEADER, VersionUtils.version),
                            Query.EMPTY, getSelf(), reference.getType(), new Callback<String>() {
                                // 通讯成功
                                @Override
                                public void onReceive(RestResult<String> result) {
                                    if (result.ok()) {
                                        // 业务成功
                                        MemberUtil.onSuccess(ServerMemberManager.this, target);
                                    } else {
                                        // 业务失败
                                        MemberUtil.onFail(ServerMemberManager.this, target);
                                    }
                                }

                                // 通讯失败
                                @Override
                                public void onError(Throwable throwable) {
                                    MemberUtil.onFail(ServerMemberManager.this, target, throwable);
                                }
                            });
        } catch (Throwable ex) {
            Loggers.CLUSTER.error("failed to report new info to target node : {}, error : {}", target.getAddress(),
                    ExceptionUtil.getAllExceptionMsg(ex));
        }
    }

    // 2s执行一次
    @Override
    protected void after() {
        GlobalExecutor.scheduleByCommon(this, 2_000L);
    }
}
```

当前节点请求其他节点/v1/core/cluster/report成功（通讯和业务都成功），执行**MemberUtil.onSuccess**方法，将对端member设置为健康，并且触发**MembersChangeEvent**事件。

```java
// MemberUtil
public static void onSuccess(final ServerMemberManager manager, final Member member) {
    final NodeState old = member.getState();
    manager.getMemberAddressInfos().add(member.getAddress());
    member.setState(NodeState.UP);
    member.setFailAccessCnt(0);
    if (!Objects.equals(old, member.getState())) {
        manager.notifyMemberChange();
    }
}
```



当前节点请求其他节点/v1/core/cluster/report失败（通讯或业务失败），执行**MemberUtil.onFail**方法，首先设置member为SUSPICIOUS状态，如果连续失败超过3次或是connection refused状态，会设置member为DOWN状态。**偶尔几次健康检查失败，不会导致member直接标记为DOWN不可用，SUSPICIOUS状态仍然会参与Distro协议，负责部分注册中心的写请求**（见后面分析DistroFilter）。最后触发**MembersChangeEvent**事件。

```java
//MemberUtil
 public static void onFail(final ServerMemberManager manager, final Member member, Throwable ex) {
     manager.getMemberAddressInfos().remove(member.getAddress());
     final NodeState old = member.getState();
     //设置member为SUSPICIOUS状态
     // 偶尔失败，设置为SUSPICIOUS中间状态，介于UP与DOWN之间，允许参与Distro协议，认为是健康节点
     member.setState(NodeState.SUSPICIOUS);
     member.setFailAccessCnt(member.getFailAccessCnt() + 1);
     int maxFailAccessCnt = EnvUtil.getProperty("nacos.core.member.fail-access-cnt", Integer.class, 3);

     // If the number of consecutive failures to access the target node reaches
     // a maximum, or the link request is rejected, the state is directly down
     // 如果连续失败超过3次或是connection refused状态，会设置member为DOWN状态
     if (member.getFailAccessCnt() > maxFailAccessCnt || StringUtils
         .containsIgnoreCase(ex.getMessage(), TARGET_MEMBER_CONNECT_REFUSE_ERRMSG)) {
         member.setState(NodeState.DOWN);
     }
     //如果member状态变化
     if (!Objects.equals(old, member.getState())) {
         manager.notifyMemberChange();
     }
 }
```

接下来看看被健康检查的对端节点如何**处理/v1/core/cluster/report请求**。

调用MemberManager的update方法。

```java
// NacosClusterController
@PostMapping(value = {"/report"})
public RestResult<String> report(@RequestBody Member node) {
    if (!node.check()) {
        return RestResultUtils.failedWithMsg(400, "Node information is illegal");
    }
    node.setState(NodeState.UP);
    node.setFailAccessCnt(0);
    boolean result = memberManager.update(node);
    return RestResultUtils.success(Boolean.toString(result));
}
```



MemberManager更新serverList中的Member信息，并发布**MembersChangeEvent**事件。这个健康检查是**双向**的，无论请求节点还是响应节点，都会更新内存中的集群节点状态。

```java
// ServerMemberManager
 public boolean update(Member newMember) {
     //ip + port
     String address = newMember.getAddress();
     // 不在配置文件中的member不会加入集群
     if (!serverList.containsKey(address)) {
         return false;
     }

     serverList.computeIfPresent(address, (s, member) -> {
         if (NodeState.DOWN.equals(newMember.getState())) {
             memberAddressInfos.remove(newMember.getAddress());
         }
         //如果基本信息发生变化
         boolean isPublishChangeEvent = MemberUtil.isBasicInfoChanged(newMember, member);
         newMember.setExtendVal(MemberMetaDataConstants.LAST_REFRESH_TIME, System.currentTimeMillis());
         MemberUtil.copy(newMember, member);
         if (isPublishChangeEvent) {
             //发布MembersChangeEvent事件
             // member basic data changes and all listeners need to be notified
             notifyMemberChange();
         }
         return member;
     });

     return true;
 }
```

总结来说，每个nacos节点会每隔2s轮询选择其他节点，上报自己的节点信息，将双方serverList中的Member信息更新。如果对端健康检查失败，对端节点标记为SUSPICIOUS，表示对端可能下线；当连续超过3次健康检查失败，会标记为对端节点为DOWN。此外，这个健康检查是双向的，每个节点都会主动发起健康检查，也会被动接收健康检查。

# 四、服务订阅/查询

**GET /nacos/v1/ns/instance/list** 服务订阅/查询

InstanceController#list

```java
@GetMapping("/list")
@Secured(parser = NamingResourceParser.class, action = ActionTypes.READ)
public ObjectNode list(HttpServletRequest request) throws Exception {
    //.....
    return doSrvIpxt(namespaceId, serviceName, agent, clusters, clientIP, udpPort, env, isCheck, app, tenant,
            healthyOnly);
}
```



**订阅与普通查询的区别是客户端传来的udpPort是否等于0**，如果等于0，表示仅查询，如果大于0表示订阅。

InstanceController#doSrvIpxt

```java
public ObjectNode doSrvIpxt(String namespaceId, String serviceName, String agent, String clusters, String clientIP,
            int udpPort, String env, boolean isCheck, String app, String tid, boolean healthyOnly) throws Exception {
  ObjectNode result = JacksonUtils.createEmptyJsonNode();
  // 1. 定位Service 根据namespace + groupName@@serviceName获取Service
  Service service = serviceManager.getService(namespaceId, serviceName);
  long cacheMillis = switchDomain.getDefaultCacheMillis();
  try {
    // udp推送服务新增一个客户端
    if (udpPort > 0 && pushService.canEnablePush(agent)) {
      pushService
        .addClient(namespaceId, serviceName, clusters, agent, new InetSocketAddress(clientIP, udpPort),
                   pushDataSource, tid, app);
      cacheMillis = switchDomain.getPushCacheMillis(serviceName); // 10s
    }
  } catch (Exception e) {
    Loggers.SRV_LOG.error();
    cacheMillis = switchDomain.getDefaultCacheMillis();
  }
  if (service == null) {
    // ...
    result.replace("hosts", JacksonUtils.createEmptyArrayNode());
    return result;
  }
  // service.enabled=false抛出异常
  checkIfDisabled(service);

  // 2. Service定位Instance
  List<Instance> srvedIPs = service.srvIPs(Arrays.asList(StringUtils.split(clusters, ",")));
  if (CollectionUtils.isEmpty(srvedIPs)) {
    // ...
    result.set("hosts", JacksonUtils.createEmptyArrayNode());
    return result;
  }
  // 对于instance分组，健康和非健康
  Map<Boolean, List<Instance>> ipMap = new HashMap<>(2);
  ipMap.put(Boolean.TRUE, new ArrayList<>());
  ipMap.put(Boolean.FALSE, new ArrayList<>());
  for (Instance ip : srvedIPs) {
    ipMap.get(ip.isHealthy()).add(ip);
  }
  // 3. 保护模式
  double threshold = service.getProtectThreshold();
  if ((float) ipMap.get(Boolean.TRUE).size() / srvedIPs.size() <= threshold) {
    ipMap.get(Boolean.TRUE).addAll(ipMap.get(Boolean.FALSE));
    ipMap.get(Boolean.FALSE).clear();
  }

  // 4. 结果组装
  ArrayNode hosts = JacksonUtils.createEmptyArrayNode();

  for (Map.Entry<Boolean, List<Instance>> entry : ipMap.entrySet()) {
    List<Instance> ips = entry.getValue();
    if (healthyOnly && !entry.getKey()) {
      continue;
    }
    for (Instance instance : ips) {
      if (!instance.isEnabled()) {
        continue;
      }
      ObjectNode ipObj = JacksonUtils.createEmptyJsonNode();
      ipObj.put("ip", instance.getIp());
      ipObj.put("port", instance.getPort());
      // ...
      hosts.add(ipObj);
    }
  }
  result.replace("hosts", hosts);
  // ...
  return result;
}
```

这段查询逻辑有点长，主要逻辑是根据namespace和group和service定位到Service实例，再根据clustername定位到Cluster，返回Cluster中的Instance列表。

ServiceManager#getService

```java
// ServiceManager.java
// namespace - groupName@@serviceName - Service
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
public Service getService(String namespaceId, String serviceName) {
    if (serviceMap.get(namespaceId) == null) {
        return null;
    }
    return chooseServiceMap(namespaceId).get(serviceName);
}
public Map<String, Service> chooseServiceMap(String namespaceId) {
  	return serviceMap.get(namespaceId);
}
// Service.java
// Cluster注册表，key是集群名称
private Map<String, Cluster> clusterMap = new HashMap<>();
public List<Instance> srvIPs(List<String> clusters) {
  if (CollectionUtils.isEmpty(clusters)) {
    clusters = new ArrayList<>();
    clusters.addAll(clusterMap.keySet());
  }
  return allIPs(clusters);
}
public List<Instance> allIPs(List<String> clusters) {
  List<Instance> result = new ArrayList<>();
  for (String cluster : clusters) {
    Cluster clusterObj = clusterMap.get(cluster);
    if (clusterObj == null) {
      continue;
    }
    result.addAll(clusterObj.allIPs());
  }
  return result;
}
```

## 客户端注册UDP监听

InstanceController#doSrvIpxt无论客户端查询的服务是否存在，都会向服务端的PushService中注册一个监听。当服务发生变化时，服务端会通过udp协议推送至客户端。这里的udpPort是客户端的udp端口号，由客户端在发起查询时传入，见上一章。



```java
@Autowired
private PushService pushService;
// InstanceController#doSrvIpxt
try {
  // udp推送服务新增一个客户端
  if (udpPort > 0 && pushService.canEnablePush(agent)) {
    pushService
      .addClient(namespaceId, serviceName, clusters, agent, new InetSocketAddress(clientIP, udpPort),
                 pushDataSource, tid, app);
    cacheMillis = switchDomain.getPushCacheMillis(serviceName); // 10s
  }
} catch (Exception e) {
  cacheMillis = switchDomain.getDefaultCacheMillis();
}
```



```java
@Component
public class PushService implements ApplicationContextAware, ApplicationListener<ServiceChangeEvent> {
    // 第一个key是namespace+groupService 第二个key是PushClient.toString
    private static ConcurrentMap<String, ConcurrentMap<String, PushClient>> clientMap = new ConcurrentHashMap<>();
}
```

另外InstanceController#doSrvIpxt控制客户端的拉取服务注册表的时间间隔为**cacheMillis=10s**，见上一章客户端服务发现**UpdateTask**。

## 保护模式

InstanceController#doSrvIpxt在处理Instance列表时有一个常见的逻辑操作。就是当某个服务下的实例**大量**下线（Instance.healthy=false）时，会开启**保护模式**，认为是服务端自己发生了网络分区，将所有实例认为是健康状态返回给客户端。这是AP模式注册中心的一个代表性功能，如Eureka。

**何为大量？**

每个Service实例中维护一个protectThreshold用于计算是否是大量服务下线，默认是0。

<img src="images/服务详情-保护阈值.jpeg" alt="服务详情-保护阈值" style="zoom:80%;" />



```java
public class Service implements Serializable {
    // 服务保护阈值，当大多数服务下线，认为当前注册中心节点发生故障，返回所有实例，包括非健康实例
    private float protectThreshold = 0.0F;
}
```

**当存活实例（ipMap.get(Boolean.TRUE).size()）/总实例（srvedIPs.size）<= protectThreshold时，认为注册中心发生故障，进入保护模式，返回服务下所有实例**。当默认配置为0时，如果某个服务下所有实例都无法与Nacos通讯，会返回该服务下所有实例。

```java
// InstanceController#doSrvIpxt
// 对于instance分组，健康和非健康
Map<Boolean, List<Instance>> ipMap = new HashMap<>(2);
ipMap.put(Boolean.TRUE, new ArrayList<>());
ipMap.put(Boolean.FALSE, new ArrayList<>());
for (Instance ip : srvedIPs) {
  ipMap.get(ip.isHealthy()).add(ip);
}
// 3. 保护模式
double threshold = service.getProtectThreshold();
if ((float) ipMap.get(Boolean.TRUE).size() / srvedIPs.size() <= threshold) {
  ipMap.get(Boolean.TRUE).addAll(ipMap.get(Boolean.FALSE));
  ipMap.get(Boolean.FALSE).clear();
}
```





# 五、写请求

对于客户端的写请求（如服务注册），对于客户端来说服务端是对等的，请求任何一个节点都可以正常响应。

但是对于服务端来说，并非所有写请求都由当前节点处理。



如/v1/ns/instance处理客户端服务注册，方法被**CanDistro注解**，此类方法都会经过**DistroFilter**。

```java
// InstanceController
@CanDistro
@PostMapping
public String register(HttpServletRequest request) throws Exception {
   //...
}
```



**ControllerMethodsCache**根据请求路径、请求方法、请求参数，找到RequestMapping注解的方法返回给DistroFilter。**DistroMapper**会根据服务名（groupName@@serviceName）定位到责任节点。

**如果当前节点是责任节点，那么继续执行后续逻辑；否则当前节点会将写请求转发至责任节点处理，然后用责任节点的响应报文来响应客户端**。

```java
public class DistroFilter implements Filter {
    @Autowired
    private DistroMapper distroMapper;
    @Autowired
    private ControllerMethodsCache controllerMethodsCache;

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {
        ReuseHttpRequest req = new ReuseHttpServletRequest((HttpServletRequest) servletRequest);
        HttpServletResponse resp = (HttpServletResponse) servletResponse;

        String urlString = req.getRequestURI();

        if (StringUtils.isNotBlank(req.getQueryString())) {
            urlString += "?" + req.getQueryString();
        }

        try {
            String path = new URI(req.getRequestURI()).getPath();
            String serviceName = req.getParameter(CommonParams.SERVICE_NAME);
            // For client under 0.8.0:
            if (StringUtils.isBlank(serviceName)) {
                serviceName = req.getParameter("dom");
            }

            if (StringUtils.isNotBlank(serviceName)) {
                serviceName = serviceName.trim();
            }
            //根据请求路径找到controller的方法
            Method method = controllerMethodsCache.getMethod(req);

            if (method == null) {
                throw new NoSuchMethodException(req.getMethod() + " " + path);
            }
            // serviceName版本适配，使用groupName@@serviceName
            String groupName = req.getParameter(CommonParams.GROUP_NAME);
            if (StringUtils.isBlank(groupName)) {
                groupName = Constants.DEFAULT_GROUP;
            }

            // use groupName@@serviceName as new service name.
            // in naming controller, will use com.alibaba.nacos.api.naming.utils.NamingUtils.checkServiceNameFormat to check it's format.
            String groupedServiceName = serviceName;
            if (StringUtils.isNotBlank(serviceName) && !serviceName.contains(Constants.SERVICE_INFO_SPLITER)) {
                groupedServiceName = groupName + Constants.SERVICE_INFO_SPLITER + serviceName;
            }

            /**
             * distro协议将节点分为责任节点和非责任节点 只有责任节点才能处理写请求（加了CanDistro注解的请求）
             */

            // 如果被CanDistro注解，且当前节点不负责groupedServiceName
            if (method.isAnnotationPresent(CanDistro.class) && !distroMapper.responsible(groupedServiceName)) {

                String userAgent = req.getHeader(HttpHeaderConsts.USER_AGENT_HEADER);

                if (StringUtils.isNotBlank(userAgent) && userAgent.contains(UtilsAndCommons.NACOS_SERVER_HEADER)) {
                    // This request is sent from peer server, should not be redirected again:
                    Loggers.SRV_LOG.error("receive invalid redirect request from peer {}", req.getRemoteAddr());
                    resp.sendError(HttpServletResponse.SC_BAD_REQUEST,
                            "receive invalid redirect request from peer " + req.getRemoteAddr());
                    return;
                }
                // 获取实际负责该服务的目标节点
                final String targetServer = distroMapper.mapSrv(groupedServiceName);
                // 组装请求参数
                List<String> headerList = new ArrayList<>(16);
                Enumeration<String> headers = req.getHeaderNames();
                while (headers.hasMoreElements()) {
                    String headerName = headers.nextElement();
                    headerList.add(headerName);
                    headerList.add(req.getHeader(headerName));
                }

                final String body = IoUtils.toString(req.getInputStream(), Charsets.UTF_8.name());
                final Map<String, String> paramsValue = HttpClient.translateParameterMap(req.getParameterMap());
                // 请求实际责任节点
                RestResult<String> result = HttpClient
                        .request("http://" + targetServer + req.getRequestURI(), headerList, paramsValue, body,
                                PROXY_CONNECT_TIMEOUT, PROXY_READ_TIMEOUT, Charsets.UTF_8.name(), req.getMethod());
                String data = result.ok() ? result.getData() : result.getMessage();
                try {
                    // 取负责节点的响应报文响应客户端
                    WebUtils.response(resp, data, result.getCode());
                } catch (Exception ignore) {
                    Loggers.SRV_LOG.warn("[DISTRO-FILTER] request failed: " + distroMapper.mapSrv(groupedServiceName)
                            + urlString);
                }
            } else {
                // 当前节点负责groupedServiceName，直接处理
                OverrideParameterRequestWrapper requestWrapper = OverrideParameterRequestWrapper.buildRequest(req);
                requestWrapper.addParameter(CommonParams.SERVICE_NAME, groupedServiceName);
                filterChain.doFilter(requestWrapper, resp);
            }
        } catch (AccessControlException e) {
            resp.sendError(HttpServletResponse.SC_FORBIDDEN, "access denied: " + ExceptionUtil.getAllExceptionMsg(e));
        } catch (NoSuchMethodException e) {
            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED,
                    "no such api:" + req.getMethod() + ":" + req.getRequestURI());
        } catch (Exception e) {
            resp.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
                    "Server failed," + ExceptionUtil.getAllExceptionMsg(e));
        }

    }
}
```

**重点在于DistroMapper如何分配哪个节点负责哪些服务**。

```java
@Component("distroMapper")
public class DistroMapper extends MemberChangeListener {
    // 健康节点
    private volatile List<String> healthyList = new ArrayList<>();
    // 开关服务
    private final SwitchDomain switchDomain;
    // 节点管理
    private final ServerMemberManager memberManager;
}
```

**DistroMapper**内部又维护了一个集群健康节点列表，当收到**MembersChangeEvent**事件时，会更新这个列表。根据第四章节的集群管理，当集群节点健康状况发生变更时，都会触发MembersChangeEvent事件。

关注onEvent方法，这里会过滤出**UP和SUSPICIOUS状态**的节点作为Distro协议认为的健康节点。

```java
//DistroMapper
public void onEvent(MembersChangeEvent event) {
    List<String> list = MemberUtil.simpleMembers(MemberUtil.selectTargetMembers(event.getMembers(),
            member -> NodeState.UP.equals(member.getState()) || NodeState.SUSPICIOUS.equals(member.getState())));
    Collections.sort(list);
    Collection<String> old = healthyList;
    healthyList = Collections.unmodifiableList(list);
}
```

**如果hash(serviceName) % healthList.size == 当前节点所处healthList下标，则认为当前节点是负责这个service的节点**。这里不是很明白为什么要indexOf+lastIndexOf共同判断，直接indexOf == target不行吗。

```java
//DistroMapper
// 健康节点
private volatile List<String> healthyList = new ArrayList<>();
public boolean responsible(String serviceName) {
    final List<String> servers = healthyList;
    // 如果关闭distro协议 或 standalone启动 认为当前节点可以负责写请求
    if (!switchDomain.isDistroEnabled() || EnvUtil.getStandaloneMode()) {
        return true;
    }
    if (CollectionUtils.isEmpty(servers)) {
        return false;
    }
    // 当前节点所处servers下标
    int index = servers.indexOf(EnvUtil.getLocalAddress());
    int lastIndex = servers.lastIndexOf(EnvUtil.getLocalAddress());
    if (lastIndex < 0 || index < 0) {
        return true;
    }
    // 哈希%servers大小
    int target = distroHash(serviceName) % servers.size();
    return target >= index && target <= lastIndex;
}
private int distroHash(String serviceName) {
  return Math.abs(serviceName.hashCode() % Integer.MAX_VALUE);
}
```

根据serviceName获取责任节点地址也是一样的逻辑。

```java
//DistroMapper
// 健康节点
private volatile List<String> healthyList = new ArrayList<>();
public String mapSrv(String serviceName) {
    final List<String> servers = healthyList;
    if (CollectionUtils.isEmpty(servers) || !switchDomain.isDistroEnabled()) {
        return EnvUtil.getLocalAddress();
    }
    try {
        int index = distroHash(serviceName) % servers.size();
        return servers.get(index);
    } catch (Throwable e) {
        return EnvUtil.getLocalAddress();
    }
}
```

# 六、服务注册

有了上述铺垫，看一下服务注册的逻辑，**POST /v1/ns/instance**。

InstanceController#register

```java
// InstanceController
@CanDistro
@PostMapping
public String register(HttpServletRequest request) throws Exception {
    final String namespaceId = WebUtils
            .optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    final String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    NamingUtils.checkServiceNameFormat(serviceName);
    final Instance instance = parseInstance(request);
    serviceManager.registerInstance(namespaceId, serviceName, instance);
    return "ok";
}
```

ServiceManager注册实例分为两步，一步是确保Service存在，第二步是将Instance加入Service。

```java
// ServiceManager
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 1. 如果首次注册，才会执行，创建Service，放入ServiceManager管理
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
    // 获取Service实例
    Service service = getService(namespaceId, serviceName);
    // 2. 把Instance加入Service
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}
```

![服务注册-流程.png](../images/31497bab12254fb19c82d8a22849ec5e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



## 创建服务

因为服务实例Instance在服务Service下面维护，优先确保ServiceManager的serviceMap中存在Service。

```java
// ServiceManager
// namespace - groupName@@serviceName - Service
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
public void createEmptyService(String namespaceId, String serviceName, boolean local) throws NacosException {
    createServiceIfAbsent(namespaceId, serviceName, local, null);
}
public void createServiceIfAbsent(String namespaceId, String serviceName, boolean local, Cluster cluster)
        throws NacosException {
    Service service = getService(namespaceId, serviceName);
    // serviceMap还没有对应service实例，才执行后续逻辑
    if (service == null) {
        service = new Service();
        service.setName(serviceName);
        service.setNamespaceId(namespaceId);
        // ...
				// 核心逻辑
        putServiceAndInit(service);
         // 非临时节点，持久化Service；临时节点不会持久化Service
        if (!local) {
            addOrReplaceService(service);
        }
    }
}
```

忽略local=false的非临时节点逻辑，重点关注**putServiceAndInit**方法。这里主要是将service写入serviceMap（内部还没有Instance），执行**Service的init方法开启服务对应客户端心跳检测**，最后**ConsistencyService**同时监听了这个服务的临时和非临时节点。

```java
// ServiceManager
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
private void putServiceAndInit(Service service) throws NacosException {
    // 1. 写入serviceMap内存
    putService(service);
    // 2. 开启客户端心跳检测
    service.init();
    // 3. 监听
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), true), service);
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), false), service);
}
```

客户端心跳检测后续再说，看看这个监听做了什么。

**ConsistencyService**一致性服务，定义了一些KV存储的常用功能，包括增删改查和监听。

```java
public interface ConsistencyService {
    void put(String key, Record value) throws NacosException;
    void remove(String key) throws NacosException;
    Datum get(String key) throws NacosException;
    void listen(String key, RecordListener listener) throws NacosException;
    void unListen(String key, RecordListener listener) throws NacosException;
    boolean isAvailable();
}
```

ConsistencyService的实现类分为两类。

一类是代理类，会根据key的pattern决定实际使用哪个ConsistencyService实现类处理。如**DelegateConsistencyServiceImpl根据key是否匹配临时节点的pattern，决定走临时节点实现还是持久节点实现**。

```java
@Service("consistencyDelegate")
public class DelegateConsistencyServiceImpl implements ConsistencyService {
    // Raft CP PersistentServiceProcessor
    private final PersistentConsistencyServiceDelegateImpl persistentConsistencyService;
    // Distro AP DistroConsistencyServiceImpl
    private final EphemeralConsistencyService ephemeralConsistencyService;
    @Override
    public void put(String key, Record value) throws NacosException {
        mapConsistencyService(key).put(key, value);
    }
    @Override
    public void listen(String key, RecordListener listener) throws NacosException {
        //...
        mapConsistencyService(key).listen(key, listener);
    }
		// ...
    private ConsistencyService mapConsistencyService(String key) {
        return KeyBuilder.matchEphemeralKey(key) ? ephemeralConsistencyService : persistentConsistencyService;
    }
}
```

另一类就是真实的实现类。

**PersistentServiceProcessor**是基于JRaft实现的一致性服务，之前看配置中心的时候知道，分为写一致和读一致（线性一致）。对于注册中心来说，**如果是持久节点会走Raft一致性服务**。

```java
// PersistentServiceProcessor
@Override
public void put(String key, Record value) throws NacosException {
    final BatchWriteRequest req = new BatchWriteRequest();
    Datum datum = Datum.createDatum(key, value);
    req.append(ByteUtils.toBytes(key), serializer.serialize(datum));
    final WriteRequest request = WriteRequest.newBuilder().setData(ByteString.copyFrom(serializer.serialize(req)))
            .setGroup(Constants.NAMING_PERSISTENT_SERVICE_GROUP).setOperation(Op.Write.desc).build();
    try {
        protocol.write(request);
    } catch (Exception e) {
        throw new NacosException(ErrorCode.ProtoSubmitError.getCode(), e.getMessage());
    }
}
@Override
public Datum get(String key) throws NacosException {
  final List<byte[]> keys = new ArrayList<>(1);
  keys.add(ByteUtils.toBytes(key));
  final ReadRequest req = ReadRequest.newBuilder().setGroup(Constants.NAMING_PERSISTENT_SERVICE_GROUP)
    .setData(ByteString.copyFrom(serializer.serialize(keys))).build();
  try {
    Response resp = protocol.getData(req);
    if (resp.getSuccess()) {
      BatchReadResponse response = serializer
        .deserialize(resp.getData().toByteArray(), BatchReadResponse.class);
      final List<byte[]> rValues = response.getValues();
      return rValues.isEmpty() ? null : serializer.deserialize(rValues.get(0), getDatumTypeFromKey(key));
    }
    throw new NacosException(ErrorCode.ProtoReadError.getCode(), resp.getErrMsg());
  } catch (Throwable e) {
    throw new NacosException(ErrorCode.ProtoReadError.getCode(), e.getMessage());
  }
}
```

**DistroConsistencyServiceImpl**基于Distro协议，如果是临时节点会走这个一致性服务，只会将数据存储在内存中。暂时先只看监听逻辑，这里传入的listener就是Service，Service实现了RecordListener接口。

```java
// DistroConsistencyServiceImpl
private Map<String, ConcurrentLinkedQueue<RecordListener>> listeners = new ConcurrentHashMap<>();
@Override
public void listen(String key, RecordListener listener) throws NacosException {
  if (!listeners.containsKey(key)) {
    listeners.put(key, new ConcurrentLinkedQueue<>());
  }
  if (listeners.get(key).contains(listener)) {
    return;
  }
  listeners.get(key).add(listener);
}
```

至此创建Service的流程走完了，主要是创建Service实例写入ServiceManager的内存map，Service开启客户端心跳检测，最后在DistroConsistencyServiceImpl注册服务实例变化的监听。

## 注册实例

服务注册的第二步是更新Service内部的Instance列表，将新的实例加入Instance列表。

```java
// ServiceManager
public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips)
            throws NacosException {

    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);

    Service service = getService(namespaceId, serviceName);

    synchronized (service) {
        // 从底层存储获取当前Instance列表，将新加入的Instance加入并返回
        List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);

        Instances instances = new Instances();
        instances.setInstanceList(instanceList);
        /**
             *  AP 临时节点 - DistroConsistencyServiceImpl#put()
             *  CP 持久节点 - PersistentServiceProcessor#put()
             */

        // 写入底层存储
        //DelegateConsistencyServiceImpl.put
        consistencyService.put(key, instances);
    }
}
```



重点看**consistencyService.put**的实现。关注临时节点注册，这里的实现类是**DistroConsistencyServiceImpl**。

```java
// DistroConsistencyServiceImpl
public void put(String key, Record value) throws NacosException {
    // 写入数据
    onPut(key, value);
    // 将写入数据，同步至所有Member
    distroProtocol.sync(new DistroKey(key, KeyBuilder.INSTANCE_LIST_KEY_PREFIX), DataOperation.CHANGE, globalConfig.getTaskDispatchPeriod() / 2);
}
```



### 本地更新

首先onPut方法将数据写入底层存储，**dataStore是一个基于内存的kv存储**，**Datum**封装了kv结构，**以服务为key，以服务下所有Instance为value，写入kv存储**。之后通过**Notifier**提交了一个任务，用于通知实例服务实例变更。



````java
// DistroConsistencyServiceImpl
private volatile Notifier notifier = new Notifier();
private final DataStore dataStore;
public void onPut(String key, Record value) {
    // 1. 如果是临时节点，写入内存map
    if (KeyBuilder.matchEphemeralInstanceListKey(key)) {
        Datum<Instances> datum = new Datum<>();
        datum.value = (Instances) value;
        datum.key = key;
        datum.timestamp.incrementAndGet();
        dataStore.put(key, datum);
    }
    if (!listeners.containsKey(key)) {
        return;
    }
    // 2. 新增key变更任务，后续通知监听器
    notifier.addTask(key, DataOperation.CHANGE);
}
````

**Notifier**是一个简单的生产消费模型实现Runnable接口，将发生变化的服务，调用对应RecordListener。

```java
// DistroConsistencyServiceImpl.Notifier
// 任务队列
private BlockingQueue<Pair<String, DataOperation>> tasks = new ArrayBlockingQueue<>(1024 * 1024);
// 生产任务
public void addTask(String datumKey, DataOperation action) {
  // ...
  tasks.offer(Pair.with(datumKey, action));
}
// 消费任务
@Override
public void run() {
  for (; ; ) {
    Pair<String, DataOperation> pair = tasks.take();
    handle(pair);
  }
}
// 调用Listener
private void handle(Pair<String, DataOperation> pair) {
    String datumKey = pair.getValue0();
    DataOperation action = pair.getValue1();
    for (RecordListener listener : listeners.get(datumKey)) {
      /**
      * 如果当前Service有新的实例加入，就把这个变更（服务列表发生变化）推送给订阅当前Service的nacos客户端
      */
      if (action == DataOperation.CHANGE) {
        listener.onChange(datumKey, dataStore.get(datumKey).value);
      }
      if (action == DataOperation.DELETE) {
        listener.onDelete(datumKey);
      }
    }
}

```

**Service** 实现RecordListener接口，当底层存储的Instance发生变更了，这里都会收到回调。更新内存中ClusterMap，并将变更后的自己的信息通过**UDP推送给监听当前服务的所有客户端**。

```java
// Service
public void onChange(String key, Instances value) throws Exception {
    // ...
    updateIPs(value.getInstanceList(), KeyBuilder.matchEphemeralInstanceListKey(key));
    // ...
}
// Cluster注册表，key是集群名称
private Map<String, Cluster> clusterMap = new HashMap<>();
public void updateIPs(Collection<Instance> instances, boolean ephemeral) {
  Map<String, List<Instance>> ipMap = new HashMap<>(clusterMap.size());
  for (Instance instance : instances) {
    // ...
  }
  for (Map.Entry<String, List<Instance>> entry : ipMap.entrySet()) {
    List<Instance> entryIPs = entry.getValue();
    clusterMap.get(entry.getKey()).updateIps(entryIPs, ephemeral);
  }
  setLastModifiedMillis(System.currentTimeMillis());
  // UDP将Service变更推送给客户端
  getPushService().serviceChanged(this);
}
```

UDP推送客户端的逻辑比较简单，参考上一章的客户端接收UDP推送逻辑 & 本章第二节服务查询时注册UDP监听客户端逻辑即可，直接跳过。

### 集群数据同步

DistroConsistencyServiceImpl写入数据到底层存储后，将写入的数据延迟1s（nacos.naming.distro.taskDispatchPeriod/2=2s/2=1s）推送给集群中所有节点。（**这意味着，客户端感知到服务注册表变更后，如果立即向集群中其他节点查询注册表，可能返回不一致数据**）

```java
// DistroConsistencyServiceImpl
public void put(String key, Record value) throws NacosException {
    // 写入数据
    onPut(key, value);
    // 将写入数据，同步至所有Member
    distroProtocol.sync(new DistroKey(key, KeyBuilder.INSTANCE_LIST_KEY_PREFIX), DataOperation.CHANGE, globalConfig.getTaskDispatchPeriod() / 2);
}
```



**DistroProtocol**循环节点中所有Member（包括不健康的），针对每个集群节点提交一个DistroDelayTask延迟任务。

```java
// DistroProtocol
 public void sync(DistroKey distroKey, DataOperation action, long delay) {
   //获取除了自己之外的其他节点(包括不健康的)
   for (Member each : memberManager.allMembersWithoutSelf()) {
     DistroKey distroKeyWithTarget = new DistroKey(distroKey.getResourceKey(), distroKey.getResourceType(),
                                                   each.getAddress());

     DistroDelayTask distroDelayTask = new DistroDelayTask(distroKeyWithTarget, action, delay);
     //添加延迟任务到队列  延迟时间是1s
     distroTaskEngineHolder.getDelayTaskExecuteEngine().addTask(distroKeyWithTarget, distroDelayTask);
     if (Loggers.DISTRO.isDebugEnabled()) {
       Loggers.DISTRO.debug("[DISTRO-SCHEDULE] {} to {}", distroKey, each.getAddress());
     }
   }
 }
```

添加延迟任务到队列

NacosDelayTaskExecuteEngine#addTask

```java
public void addTask(Object key, AbstractDelayTask newTask) {
    lock.lock();
    try {
        AbstractDelayTask existTask = tasks.get(key);
        if (null != existTask) {
            newTask.merge(existTask);
        }
        // key = groupKey, Task = DumpTask
        tasks.put(key, newTask);
    } finally {
        lock.unlock();
    }
}

protected void processTasks() {
  Collection<Object> keys = getAllTaskKeys();
  for (Object taskKey : keys) {
    // 获取groupKey对应的DumpTask
    AbstractDelayTask task = removeTask(taskKey);
    if (null == task) {
      continue;
    }
    // 通过groupKey找到对应的task处理器（map管理）
    NacosTaskProcessor processor = getProcessor(taskKey);
    if (null == processor) {
      getEngineLog().error("processor not found for task, so discarded. " + task);
      continue;
    }
    try {
      // ReAdd task if process failed
      // 处理器处理Task DumpTask ->  DumpProcessor.process
      //  DistroDelayTask(服务注册 同步到其他节点的) ->  DistroDelayTaskProcessor.process
      if (!processor.process(task)) {
        //如果处理失败（比如为了更新本地文件系统，获取写锁失败），把任务重新添加到队列中（重试）
        retryFailedTask(taskKey, task);
      }
    } catch (Throwable e) {
      getEngineLog().error("Nacos task execute error : " + e.toString(), e);
      retryFailedTask(taskKey, task);
    }
  }
}
```

DistroDelayTaskProcessor#process



```java
public boolean process(NacosTask task) {
    if (!(task instanceof DistroDelayTask)) {
        return true;
    }
    DistroDelayTask distroDelayTask = (DistroDelayTask) task;
    DistroKey distroKey = distroDelayTask.getDistroKey();
    if (DataOperation.CHANGE.equals(distroDelayTask.getAction())) {
        DistroSyncChangeTask syncChangeTask = new DistroSyncChangeTask(distroKey, distroComponentHolder);
      	//添加同步数据任务到队列
        distroTaskEngineHolder.getExecuteWorkersManager().addTask(distroKey, syncChangeTask);
        return true;
    }
    return false;
}
```

添加同步数据的任务到队列，最终是调用DistroSyncChangeTask.run()方法进行数据同步

DistroSyncChangeTask#run

```java
public void run() {
    Loggers.DISTRO.info("[DISTRO-START] {}", toString());
    try {
        String type = getDistroKey().getResourceType();
      	//通过key重新查询底层存储获取最新数据
        DistroData distroData = distroComponentHolder.findDataStorage(type).getDistroData(getDistroKey());
        distroData.setType(DataOperation.CHANGE);
      	//DistroHttpAgent#syncData()
        boolean result = distroComponentHolder.findTransportAgent(type).syncData(distroData, getDistroKey().getTargetServer());
        if (!result) {
            handleFailedTask();
        }
        Loggers.DISTRO.info("[DISTRO-END] {} result: {}", toString(), result);
    } catch (Exception e) {
        Loggers.DISTRO.warn("[DISTRO] Sync data change failed.", e);
        handleFailedTask();
    }
}
```

DistroHttpAgent#syncData()

```java
public boolean syncData(DistroData data, String targetServer) {
  if (!memberManager.hasMember(targetServer)) {
    return true;
  }
  byte[] dataContent = data.getContent();
  return NamingProxy.syncData(dataContent, data.getDistroKey().getTargetServer());
}
```

NamingProxy#syncData

```java
public static boolean syncData(byte[] data, String curServer) {
    Map<String, String> headers = new HashMap<>(128);

  
    try {
        //   请求/v1/ns/distro/datum
        RestResult<String> result = HttpClient.httpPutLarge(
                "http://" + curServer + EnvUtil.getContextPath() + UtilsAndCommons.NACOS_NAMING_CONTEXT
                        + DATA_ON_SYNC_URL, headers, data);
        if (result.ok()) {
            return true;
        }
        if (HttpURLConnection.HTTP_NOT_MODIFIED == result.getCode()) {
            return true;
        }
        
    } catch (Exception e) {
        Loggers.SRV_LOG.warn("NamingProxy", e);
    }
    return false;
}
```

DistroController#onSyncDatum处理/v1/ns/distro/datum请求

```java
public ResponseEntity onSyncDatum(@RequestBody Map<String, Datum<Instances>> dataMap) throws Exception {

    if (dataMap.isEmpty()) {
        Loggers.DISTRO.error("[onSync] receive empty entity!");
        throw new NacosException(NacosException.INVALID_PARAM, "receive empty entity!");
    }

    for (Map.Entry<String, Datum<Instances>> entry : dataMap.entrySet()) {
        if (KeyBuilder.matchEphemeralInstanceListKey(entry.getKey())) {
            String namespaceId = KeyBuilder.getNamespace(entry.getKey());
            String serviceName = KeyBuilder.getServiceName(entry.getKey());
            if (!serviceManager.containService(namespaceId, serviceName) && switchDomain
                    .isDefaultInstanceEphemeral()) {
              	//如果首次注册，才会执行，创建Service，放入ServiceManager管理
                serviceManager.createEmptyService(namespaceId, serviceName, true);
            }
            DistroHttpData distroHttpData = new DistroHttpData(createDistroKey(entry.getKey()), entry.getValue());
            distroProtocol.onReceive(distroHttpData);
        }
    }
    return ResponseEntity.ok("ok");
}
```

DistroProtocol#onReceive

```java
public boolean onReceive(DistroData distroData) {
    String resourceType = distroData.getDistroKey().getResourceType();
    DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
    if (null == dataProcessor) {
        Loggers.DISTRO.warn("[DISTRO] Can't find data process for received data {}", resourceType);
        return false;
    }
    return dataProcessor.processData(distroData);
}
```

DistroConsistencyServiceImpl#processData()

是调用DistroConsistencyServiceImpl的**processData**方法，转换参数后还是调用了本地更新方法onPut，参考上一小节。

```java
// DistroConsistencyServiceImpl
public boolean processData(DistroData distroData) {
    DistroHttpData distroHttpData = (DistroHttpData) distroData;
    Datum<Instances> datum = (Datum<Instances>) distroHttpData.getDeserializedContent();
    onPut(datum.key, datum.value);
    return true;
}

```









# 七、客户端心跳

## 心跳超时检测任务

服务注册时，**每个Service都会开启一个定时任务，用于检测当前服务下的Instance是否按时发送过心跳**。定时任务在Service的init方法调用时开启，**每5s执行一次**。



```java
// Service
private ClientBeatCheckTask clientBeatCheckTask = new ClientBeatCheckTask(this);
// Cluster注册表，key是集群名称
private Map<String, Cluster> clusterMap = new HashMap<>();
public void init() {
    // 提交客户端心跳检测任务 每5s执行一次
    HealthCheckReactor.scheduleCheck(clientBeatCheckTask);
    // ...
}
```

**ClientBeatCheckTask**客户端心跳超时检测任务，循环所有临时节点，**如果超过15秒没有收到心跳**，标记Instance为非健康（**注意DataStore没有更新**）；**如果超过30秒没有收到心跳**，调用当前节点DELETE /v1/ns/instance删除服务下的这个实例。（DELETE /v1/ns/instance流程和注册实例相反，但是也是更新Service下的Instance）

```java
public class ClientBeatCheckTask implements Runnable {
    private Service service;
    @Override
    public void run() {
        try {
            // distro 如果当前节点不负责这个service，不处理
            if (!getDistroMapper().responsible(service.getName())) {
                return;
            }
            if (!getSwitchDomain().isHealthCheckEnabled()) {
                return;
            }
            // 所有临时实例
            List<Instance> instances = service.allIPs(true);
            // 超过15s没收到心跳，标记为非健康
            for (Instance instance : instances) {
                
                if (System.currentTimeMillis() - instance.getLastBeat() > instance.getInstanceHeartBeatTimeOut()) {
                    if (!instance.isMarked()) {
                        if (instance.isHealthy()) {
                            instance.setHealthy(false);
                            // UDP推送客户端
                            getPushService().serviceChanged(service);
                        }
                    }
                }
            }
            if (!getGlobalConfig().isExpireInstance()) {
                return;
            }
            // 超过30s没收到心跳，直接从注册表中删除
            for (Instance instance : instances) {
                if (instance.isMarked()) {
                    continue;
                }
                if (System.currentTimeMillis() - instance.getLastBeat() > instance.getIpDeleteTimeout()) {
                    // 走http调用当前节点 DELETE /v1/ns/instance
                    deleteIp(instance);
                }
            }

        } catch (Exception e) {
            Loggers.SRV_LOG.warn("Exception while processing client beat time out.", e);
        }
    }
}

```



心跳超时的两个阈值是从Instance的metadata中来的，**preserved.heart.beat.timeout（\**默认15s）和\**preserved.ip.delete.timeout**（默认30s），单位都是毫秒。

```java
// Instance
public long getInstanceHeartBeatTimeOut() {
    return getMetaDataByKeyWithDefault(PreservedMetadataKeys.HEART_BEAT_TIMEOUT,
            Constants.DEFAULT_HEART_BEAT_TIMEOUT);
}

public long getIpDeleteTimeout() {
    return getMetaDataByKeyWithDefault(PreservedMetadataKeys.IP_DELETE_TIMEOUT,
            Constants.DEFAULT_IP_DELETE_TIMEOUT);
}
```



实例元数据可以在服务注册的时候设置，也可以在控制台设置，可以是**json**格式，也可以是**k1=v1,k2=v2**形式。

![控制台-实例元数据设置.png](../images/1d6de47248e84a2d8ffdfe7cc6bd995b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



## 处理客户端心跳

客户端心跳**PUT /nacos/v1/ns/instance/beat**。由于客户端心跳是个写操作（更新内存中的实例上次心跳时间），所以被@CanDistro注解，由集群中责任节点处理。



```java
// com.alibaba.nacos.naming.controllers.InstanceController
@CanDistro
@PutMapping("/beat")
public ObjectNode beat(HttpServletRequest request) throws Exception {
    ObjectNode result = JacksonUtils.createEmptyJsonNode();
    // 默认控制客户端心跳是5秒
    result.put(SwitchEntry.CLIENT_BEAT_INTERVAL, switchDomain.getClientBeatInterval());
    // 从请求获取clientBeat,namespace,serviceName,clusterName...
   	String beat = WebUtils.optional(request, "beat", StringUtils.EMPTY);
    RsInfo clientBeat = null;
    if (StringUtils.isNotBlank(beat)) {
      clientBeat = JacksonUtils.toObj(beat, RsInfo.class);
    }
    Instance instance = serviceManager.getInstance(namespaceId, serviceName, clusterName, ip, port);
    // 1. 如果服务没注册，执行注册逻辑
    if (instance == null) {
        // 如果lightBeatEnabled=true且发送心跳时客户端没有注册，需要客户端发起注册
        if (clientBeat == null) {
            result.put(CommonParams.CODE, NamingResponseCode.RESOURCE_NOT_FOUND);
            return result;
        }
        // 服务端注册逻辑
        instance = new Instance();
        instance.setPort(clientBeat.getPort());
        instance.setIp(clientBeat.getIp());
        // ...
        serviceManager.registerInstance(namespaceId, serviceName, instance);
    }
    Service service = serviceManager.getService(namespaceId, serviceName);
    if (clientBeat == null) {
        clientBeat = new RsInfo();
        clientBeat.setIp(ip);
       // ...
    }
    // 2. 更新instance健康状态
    service.processClientBeat(clientBeat);
    result.put(CommonParams.CODE, NamingResponseCode.OK);
    // 如果instance有设置心跳间隔preserved.heart.beat.interval，优先使用instance设置的心跳间隔
    if (instance.containsMetadata(PreservedMetadataKeys.HEART_BEAT_INTERVAL)) {
        result.put(SwitchEntry.CLIENT_BEAT_INTERVAL, instance.getInstanceHeartBeatInterval());
    }
    // 服务端控制是否允许light beat, 默认true
    result.put(SwitchEntry.LIGHT_BEAT_ENABLED, switchDomain.isLightBeatEnabled());
    return result;
}
```

**客户端发送心跳时，服务端没有收到客户端服务注册请求怎么处理？**

回顾上一章，当客户端发送心跳后，判断服务端响应**code=NamingResponseCode.RESOURCE_NOT_FOUND**时，会主动发起一次注册请求。

但是这取决于**lightBeatEnabled**。

lightBeatEnabled=false，表示不允许light beat，需要客户端发送全量的心跳信息。当服务端发现客户端尚未注册时，会使用请求参数中的beat反序列化为RsInfo，代替客户端进行注册。

**lightBeatEnabled=true，是默认选项**，表示客户端不会发送完整的心跳信息。当服务端发现客户端尚未注册时会返回code=NamingResponseCode.RESOURCE_NOT_FOUND。

总结，如上一章所述，默认情况下客户端发送心跳时，服务端没有收到客户端服务注册请求，需要客户端发起注册请求。

**心跳间隔到底是多少？**

上一章提到过，默认情况下客户端每**5s**发起一次心跳请求。

从服务端看，心跳间隔和心跳超时阈值一样，可以通过配置Instance元数据控制，key=**preserved.heart.beat.interval**，服务端的默认时长也是**5s**。

```java
// Instance
public long getInstanceHeartBeatInterval() {
    return getMetaDataByKeyWithDefault(PreservedMetadataKeys.HEART_BEAT_INTERVAL,
            Constants.DEFAULT_HEART_BEAT_INTERVAL);
}
```

**如何处理心跳？**

客户端心跳交由Service处理，传入RsInfo包含客户端ip、port、cluster等关键信息。

```java
// Service
public void processClientBeat(final RsInfo rsInfo) {
    ClientBeatProcessor clientBeatProcessor = new ClientBeatProcessor();
    clientBeatProcessor.setService(this);
    clientBeatProcessor.setRsInfo(rsInfo);
    HealthCheckReactor.scheduleNow(clientBeatProcessor);
}
```

提交一个**ClientBeatProcessor**任务异步立即执行，更新心跳对应Instance的健康状况和上次心跳时间。如果健康状况变更，udp通知有监听的客户端。

```java
// ClientBeatProcessor
public void run() {
  Service service = this.service;
  String ip = rsInfo.getIp();
  String clusterName = rsInfo.getCluster();
  int port = rsInfo.getPort();
  Cluster cluster = service.getClusterMap().get(clusterName);
  // 获取所有临时节点
  List<Instance> instances = cluster.allIPs(true);
  for (Instance instance : instances) {
    // 找到心跳对应实例
    if (instance.getIp().equals(ip) && instance.getPort() == port) {
      // 更新上次心跳时间
      instance.setLastBeat(System.currentTimeMillis());
      if (!instance.isMarked()) {
        // 更新健康状况
        if (!instance.isHealthy()) {
          instance.setHealthy(true);
          // udp推送客户端
          getPushService().serviceChanged(service);
        }
      }
    }
  }
}
```



注意到这里并没有用ConsistencyService更新底层存储的Instance（**注意DataStore没有更新**）。

# 八、集群数据同步

无论是服务注册POST /v1/ns/instance还是服务注销DELETE /v1/ns/instance，责任节点都会异步将变更注册信息同步至其他非责任节点。

但是在心跳的处理上，**实例健康状况变更，集群间数据并没有同步**(30s心跳超时摘除Instance也走了DELETE /v1/ns/instance，这个情况除外)。甚至**责任节点的ServiceManager与DataStore中Instance的健康状态都不一致**。这是为什么？

**对于集群数据不一致**。15s超时表示短暂网络抖动，不认为服务实例真正下线了，暂时不需要同步给其他节点，减少频繁数据同步。只有当30s超时，认为实例真的下线了，从Service里真实摘除后，通过DELETE /v1/ns/instance真实执行服务注销流程（更新ServiceManager/DataStore/集群同步）。

**对于节点内部数据不一致**。所有节点对外提供的查询服务接口，都是走的ServiceManager，Service里的Instance是会设置健康状态为false的，对客户端来说是正确的。DataStore属于Nacos内部逻辑，集群数据同步用的。



另外，为了保证所有节点的数据一致，实际上在DistroProtocol构造时，会提交一个定时任务**DistroVerifyTask**，**责任节点每5s向其他节点同步全量同步自己负责的服务信息**。这个DistroData的类型是**VERIFY**，数据只包含服务包含的Instance列表的摘要信息（MD5）。

```java
public class DistroVerifyTask implements Runnable {
    private final ServerMemberManager serverMemberManager;
    private final DistroComponentHolder distroComponentHolder;
    
    @Override
    public void run() {
        List<Member> targetServer = serverMemberManager.allMembersWithoutSelf();
        for (String each : distroComponentHolder.getDataStorageTypes()) {
          verifyForDataStorage(each, targetServer);
        }
    }
    
    private void verifyForDataStorage(String type, List<Member> targetServer) {
        DistroData distroData = distroComponentHolder.findDataStorage(type).getVerifyData();
        distroData.setType(DataOperation.VERIFY);
        for (Member member : targetServer) {
          distroComponentHolder.findTransportAgent(type).syncVerifyData(distroData, member.getAddress());
        }
    }
}
```



其他非责任节点通过**PUT /v1/ns/distro/checksum**接收VERIFY Distro数据。

非责任节点DistroConsistencyServiceImpl#onReceiveChecksums结合当前节点DataStore中的数据，比对出需要更新和需要删除的服务。

**对需要删除的服务**，从DataSore和ServiceManager中删除。

**对需要更新的服务**，需要调用GET /v1/ns/distro/datum反查查询责任节点获取服务对应注册表信息（从DataStore中查询），更新DataStore和ServiceManager中的注册信息。

```java
// DistroConsistencyServiceImpl
public void onReceiveChecksums(Map<String, String> checksumMap, String server) {
  // 根据责任节点发来的数据，结合自己DataStore里的数据分组
  // 待更新的service
  List<String> toUpdateKeys = new ArrayList<>();
  // 待删除的service
  List<String> toRemoveKeys = new ArrayList<>();

  // 删除服务 dataStore&ServiceManager
  for (String key : toRemoveKeys) {
    onRemove(key);
  }

  // 更新服务，二次请求GET /v1/ns/distro/datum获取，更新dataStore&ServiceManager内存注册表
  DistroHttpCombinedKey distroKey = new DistroHttpCombinedKey(KeyBuilder.INSTANCE_LIST_KEY_PREFIX, server);
  distroKey.getActualResourceTypes().addAll(toUpdateKeys);
  DistroData remoteData = distroProtocol.queryFromRemote(distroKey);
  if (null != remoteData) {
    processData(remoteData.getContent());
  }
}
```

# 总结

- 服务端模型主要包括：ServiceManager管理namespace+group+service到Service实例的映射关系；Service管理服务与Cluster集群的映射关系；Cluster管理其下面的持久/临时Instance列表。

  ![命名服务实现模型.png](../images/dfcc1b19ca434f4ba8f2186bb7197d72~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



- **Nacos集群管理**：Nacos集群通过MemberLoopup初始化，一般可以用nacos.home/cluster/cluster.conf配置文件的方式初始化。**每个节点每2s执行POST /v1/core/cluster/report**，向集群随机节点（包含DOWN）发送当前节点信息，一方面是为了**同步当前节点信息**，另一方面也是**健康检查**。

健康检查是双向的，每个节点都会主动发起健康检查，也会被动接收健康检查。如果健康检查失败，对端节点标记为**SUSPICIOUS**，表示对端可能下线，但是可以参与Distro协议负责处理写请求；当**连续超过3次健康检查失败**，会标记为对端节点为**DOWN**。

- **Distro写请求**：对于客户端写请求，如服务注册、客户端心跳，DistroFilter会拦截（基于@CanDistro注解）。判断请求参数中的groupServiceName是否属于当前节点管理范围（通过hash取模），如果不属于当前节点管理，转发其他节点处理，用其他节点的返回信息返回客户端；如果属于当前节点管理，直接进入Controller处理。

- **服务订阅/查询**：根据Distro协议，集群**每个节点都存储所有数据**，**每个节点都可以处理读请求**，返回当前节点注册表里的数据，无论数据是否是最新的。服务查询是根据客户端提供的namespace、groupServiceName定位到服务端ServiceManager管理的Service。另外**1、如果udpPort大于0，执行服务订阅，会注册客户端UDP监听信息到内存Map，待注册表变更后通过UDP通知客户端；2、当某个服务下的实例大量（通过service.protectThreshold控制，默认0）下线时，会开启保护模式。服务端认为自己发生了网络分区，将所有实例认为是健康状态返回给客户端**。

- **服务注册**：会经过DistroFilter，只能由责任节点处理。主要做了三件事情：1、更新当前节点的注册表（内存Map）；2、将更新后的Service同步至其他节点（Distro协议规定每个节点都能执行查询请求，每个节点都有全量数据）；3、UDP推送监听该服务的客户端。

- **客户端心跳**：会经过DistroFilter。客户端**每5s向服务端发起心跳**请求**PUT /nacos/v1/ns/instance/beat**，服务端会更新内存中Instance的上次心跳时间和健康状况；服务端每个Service，**每5s执行一次心跳超时检测任务**，如果Instance超过**15秒没有发送过心跳，设置Instance非健康**，如果Instance**超过30秒没有发送过心跳，直接摘除Instance**。心跳相关时间配置，可以通过控制台或服务注册时设置在Instance的metadata中。

| 含义                                   | Instance元数据Key             | 默认值 |
| -------------------------------------- | ----------------------------- | ------ |
| 客户端发送心跳间隔（毫秒）             | preserved.heart.beat.interval | 5000   |
| 心跳超时时间（标记实例非健康）（毫秒） | preserved.heart.beat.timeout  | 15000  |
| 心跳超时时间（摘除实例）（毫秒）       | preserved.ip.delete.timeout   | 30000  |

- **集群数据同步**：**当发生服务注册或服务注销（包含客户端30s心跳超时）**时，责任节点会将服务数据同步至其他非责任节点。**当服务端检测到客户端心跳15s超时（不满30s）**，只会在当前责任节点标记实例为非健康状态，不会将非健康状态同步至其他节点；**当服务端重新接收到客户端心跳后（15-30s中间）**，重新标记实例为健康，也不会做数据同步。**责任节点每5秒（默认nacos.core.protocol.distro.data.verify_interval_ms=5000ms），同步所有自己负责Service的Instance列表的MD5到其他节点，其他节点如果发现MD5发生变更，会反查责任节点，然后更新本地数据**。

  

  





