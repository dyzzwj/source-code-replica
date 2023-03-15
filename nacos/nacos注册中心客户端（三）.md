​																				nacos注册中心客户端

- 服务注册：客户端如何进行服务注册，服务端如何感知客户端存活
- 服务注销：服务下架
- 服务发现：服务查询是读本地内存中的注册表，还是读服务端的实时注册表，如果是读本地内存表如何保证本地内存表数据正确；订阅服务，如何感知对应服务的实例列表更新





# 一、使用案例

这里直接使用官方提供的案例，Nacos作为注册中心对于客户端来说要满足几个需求：

- 服务注册
- 服务注销
- 服务发现

```java
public class NamingExample {

    public static void main(String[] args) throws NacosException, IOException {
        // 创建命名服务
        Properties properties = new Properties();
        properties.setProperty("serverAddr", "localhost");
        NamingService naming = NamingFactory.createNamingService(properties);
        // 服务注册
        naming.registerInstance("nacos.test.3", "11.11.11.11", 8888, "TEST1");
        naming.registerInstance("nacos.test.3", "2.2.2.2", 9999, "DEFAULT");
        // 服务发现：根据服务名称 获取服务实例列表
        System.out.println(naming.getAllInstances("nacos.test.3"));
        // 服务注销
        naming.deregisterInstance("nacos.test.3", "2.2.2.2", 9999, "DEFAULT");
        // 服务发现：根据服务名称 获取服务实例列表
        System.out.println(naming.getAllInstances("nacos.test.3"));
        // 服务发现：监听服务变更
        Executor executor = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r);
                        thread.setName("test-thread");
                        return thread;
                    }
                });

        naming.subscribe("nacos.test.3", new AbstractEventListener() {
            @Override
            public Executor getExecutor() {
                return executor;
            }

            @Override
            public void onEvent(Event event) {
                System.out.println(((NamingEvent) event).getServiceName());
                System.out.println(((NamingEvent) event).getInstances());
            }
        });
    }
}
```

# 二、模型

![服务分组.png](../images/12781d0e74a84ef799b8a53c452d0afd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306220405815.awebp)

**Namspace（Tenant）：命名空间（租户）**，默认命名空间是public。一个命名空间可以包含多个Group。

![控制台-命名空间.png](../images/ae6397c28fd64ba49da45339589960c2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306220356545.awebp)

**Group：组**，默认分组是DEFAULT_GROUP。

**Service：应用服务**。

![控制台-1.png](../images/35a2f025081f4a6a9e645743cc4916f8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**Cluster**：**集群**，默认集群是DEFAULT。

**Instance：服务实例**。

![控制台-2.png](../images/3acc528b8d6a4fab81b83dbb7cac37cb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**Service和Cluster和Instance的关系**，在客户端都以**ServiceInfo**表示，由分组+集群+服务确定唯一。

**namespace信息在NacosNamingService中**

```java
public class ServiceInfo {
    // 服务名
    private String name;
    // 分组名
    private String groupName;
    // 集群，逗号分割
    private String clusters;
    // Instance实例
    private List<Instance> hosts = new ArrayList<Instance>();
}
```

**Instance实例**，**由分组+集群+服务+ip+端口确定唯一**。



```java
public class Instance {
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



# 三、NacosNamingService

NacosNamingService，NamingService的实现类，Nacos命名服务。实现了所有命名服务需要具备的功能。

以下是NacosNamingService的成员变量。

```java
public class NacosNamingService implements NamingService {
    // 命名空间/租户 默认public
    private String namespace;
    // 类似于apollo的meta-server 提供nacos集群地址，如果启用会无视serverList
    private String endpoint;
    // nacos集群地址列表，逗号分割，用户传入
    private String serverList;
    // 本地缓存路径
    private String cacheDir;
    // 日志的文件名
    private String logName;
    // 服务监听/服务注册表缓存/服务查询
    private HostReactor hostReactor;
    // 心跳维护
    private BeatReactor beatReactor;
    // 命名服务代理
    private NamingProxy serverProxy;
}
```



## 1、服务注册

服务注册逻辑在NacosNamingService的**registerInstance**方法中。

```java
// NacosNamingService
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);
    // 对于nacos来说，serviceName = groupName + @@ + serviceName
   /**
    * 根据分组groupName和应用serviceName生成了Nacos注册中心里实际的serviceName服务名。
    * Nacos服务名=groupName + @@ + serviceName。
    */
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);

    /**
         * 1、临时实例注册，默认，Instance.ephemeral=true，由客户端主动发起心跳来维护服务端注册表。
         * 如果15s内没有收到客户端心跳，服务端会设置实例为非健康；如果30s内没有收到客户端心跳，服务端将实例从注册表中摘除。服务端使用Distro协议，AP。（下一章再看Distro）
         *
         * 2、持久实例注册，Instance.ephemeral=false，由服务端主动对客户端做健康检查，
         * 默认情况下是通过TCP探测客户端存活，永远不会摘除实例，只会标记为非健康。服务端使用Raft协议，CP。
         */
    // 1. 如果是临时实例，开启心跳任务
    if (instance.isEphemeral()) {
        BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
        beatReactor.addBeatInfo(groupedServiceName, beatInfo);
    }
    // 2. POST /nacos/v1/ns/instance
    serverProxy.registerService(groupedServiceName, groupName, instance);
}
```

关注**NamingUtils.getGroupedName**方法，这个方法根据分组groupName和应用serviceName生成了Nacos注册中心里实际的serviceName服务名。**Nacos服务名=groupName + @@ + serviceName**。

```java
public static String getGroupedName(final String serviceName, final String groupName) {
    if (StringUtils.isBlank(serviceName)) {
        throw new IllegalArgumentException("Param 'serviceName' is illegal, serviceName is blank");
    }
    final String resultGroupedName = groupName + Constants.SERVICE_INFO_SPLITER + serviceName;
    return resultGroupedName.intern();
}
```



Nacos服务注册有两个大的分支逻辑：

- **临时实例注册**，默认，Instance.ephemeral=true，由客户端主动发起心跳来维护服务端注册表。如果15s内没有收到客户端心跳，服务端会设置实例为非健康；如果30s内没有收到客户端心跳，服务端将实例从注册表中摘除。服务端使用Distro协议，AP。（下一章再看Distro）
- **持久实例注册**，Instance.ephemeral=false，由服务端主动对客户端做健康检查，默认情况下是通过TCP探测客户端存活，永远不会摘除实例，只会标记为非健康。服务端使用Raft协议，CP。

这里只考虑临时实例注册，这是默认情况。总体分为两个步骤：**开启心跳任务**，**向Nacos服务端发送注册请求**。

### 心跳任务

客户端需要开启心跳任务**BeatTask**，具体逻辑在**BeatReactor**中。

首先看一下**BeatInfo**。BeatInfo的大多数成员变量来源于Instance。

```java
// BeatReactor
public BeatInfo buildBeatInfo(String groupedServiceName, Instance instance) {
    BeatInfo beatInfo = new BeatInfo();
    beatInfo.setServiceName(groupedServiceName);
    beatInfo.setIp(instance.getIp());
    beatInfo.setPort(instance.getPort());
    beatInfo.setCluster(instance.getClusterName());
    beatInfo.setWeight(instance.getWeight());
    beatInfo.setMetadata(instance.getMetadata());
    beatInfo.setScheduled(false);
    beatInfo.setPeriod(instance.getInstanceHeartBeatInterval());
    return beatInfo;
}
```



BeatReactor的**addBeatInfo**方法，将**BeatInfo**封装到一个**BeatTask**中提交到线程池中，执行一个延迟任务。

```java
// BeatReactor
// 用于执行客户端心跳任务的线程池
private final ScheduledExecutorService executorService;
// BeatInfo唯一标识 - BeatInfo
public final Map<String, BeatInfo> dom2Beat = new ConcurrentHashMap<String, BeatInfo>();

public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
        
        //心跳key 关注buildKey方法，构造每个BeatInfo的唯一标识=groupName + @@ + serviceName + # + ip + # + port。
        String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
        BeatInfo existBeat = null;
        //fix #1733
        // 关闭已经存在心跳任务
        if ((existBeat = dom2Beat.remove(key)) != null) {
            existBeat.setStopped(true);
        }
        dom2Beat.put(key, beatInfo);
        // #2 延迟任务 BeatInfo.period决定了默认客户端心跳间隔，默认间隔为5s，来源于Instance的metadata，key是preserved.heart.beat.interval。
        executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
        MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
    }
```



关注buildKey方法，构造每个**BeatInfo的唯一标识=groupName + @@ + serviceName + # + ip + # + port**。

```java
// BeatReactor
public String buildKey(String serviceName, String ip, int port) {
    return serviceName + Constants.NAMING_INSTANCE_ID_SPLITTER + ip + Constants.NAMING_INSTANCE_ID_SPLITTER + port;
}
```

**BeatTask**心跳任务的流程如下。

```java
// BeatReactor
// 命名服务代理
private final NamingProxy serverProxy;
// 向服务端发送心跳报文，是否需要包含所有BeatInfo信息
private boolean lightBeatEnabled = false;
// BeatReactor.BeatTask
class BeatTask implements Runnable {

    BeatInfo beatInfo;

    public BeatTask(BeatInfo beatInfo) {
        this.beatInfo = beatInfo;
    }

    @Override
     public void run() {
         // 0. isStopped控制心跳任务停止
         if (beatInfo.isStopped()) {
             return;
         }
         long nextTime = beatInfo.getPeriod();
         try {
             // 发送心跳请求给服务端 /nacos/v1/ns/instance/beat。
             JsonNode result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
             // 1. 服务端可以决定客户端的心跳间隔
             long interval = result.get("clientBeatInterval").asLong();
             // 2. 服务端可以决定客户端是否要发送所有BeatInfo信息
             boolean lightBeatEnabled = false;
             if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
                 lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
             }
             // 向服务端发送心跳报文，是否需要包含所有BeatInfo信息
             BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
             if (interval > 0) {
                 nextTime = interval;
             }
             int code = NamingResponseCode.OK;
             if (result.has(CommonParams.CODE)) {
                 code = result.get(CommonParams.CODE).asInt();
             }
             // 3. 如果当前Instance在服务端没找到，尝试注册
             //服务端找不到客户端实例，如何处理心跳
             //如果客户端向服务端发送心跳时，服务端没有在注册表中找到对应客户端实例，客户端会根据服务端响应状态码RESOURCE_NOT_FOUND（20404）做特殊处理，会尝试向服务端发起一次注册请求。
             if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
                 Instance instance = new Instance();
                 instance.setPort(beatInfo.getPort());
                 instance.setIp(beatInfo.getIp());
                 instance.setWeight(beatInfo.getWeight());
                 instance.setMetadata(beatInfo.getMetadata());
                 instance.setClusterName(beatInfo.getCluster());
                 instance.setServiceName(beatInfo.getServiceName());
                 instance.setInstanceId(instance.getInstanceId());
                 instance.setEphemeral(true);
                 try {

                     serverProxy.registerService(beatInfo.getServiceName(),
                                                 NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
                 } catch (Exception ignore) {
                 }
             }
         } catch (NacosException ex) {
             NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                                 JacksonUtils.toJson(beatInfo), ex.getErrCode(), ex.getErrMsg());

         }
         // 4. 提交下一次心跳任务
         executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
     }
}
```



关注几个点。

#### 心跳间隔

**BeatInfo.period**决定了默认客户端心跳间隔，**默认间隔为5s**，来源于Instance的metadata，key是**preserved.heart.beat.interval**。

```java
// Instance
private Map<String, String> metadata = new HashMap<String, String>();
public long getInstanceHeartBeatInterval() {
    return getMetaDataByKeyWithDefault(PreservedMetadataKeys.HEART_BEAT_INTERVAL, TimeUnit.SECONDS.toMillis(5));
}
private long getMetaDataByKeyWithDefault(final String key, final long defaultValue) {
    if (getMetadata() == null || getMetadata().isEmpty()) {
      return defaultValue;
    }
    final String value = getMetadata().get(key);
    if (!StringUtils.isEmpty(value) && value.matches(NUMBER_PATTERN)) {
      return Long.parseLong(value);
    }
    return defaultValue;
}
```

此外，服务端心跳响应报文中**clientBeatInterval**可以指定客户端的心跳频率。

```java
// BeatReactor.BeatTask
// 1. 服务端可以决定客户端的心跳间隔
long interval = result.get("clientBeatInterval").asLong();
if (interval > 0) {
  nextTime = interval;
}
```

#### 心跳请求

**NamingProxy** 是命名服务代理，封装了一些方法请求Nacos服务端。心跳请求报文无非是要告诉Nacos服务端，哪个服务新增了哪个ip:port对应的实例。请求地址**PUT /nacos/v1/ns/instance/beat**。

NamingProxy#sendBeat

```java
public JsonNode sendBeat(BeatInfo beatInfo, boolean lightBeatEnabled) throws NacosException {

    if (NAMING_LOGGER.isDebugEnabled()) {
        NAMING_LOGGER.debug("[BEAT] {} sending beat to server: {}", namespaceId, beatInfo.toString());
    }
    Map<String, String> params = new HashMap<String, String>(8);
    Map<String, String> bodyMap = new HashMap<String, String>(2);
    //注意到lightBeatEnabled参数用于控制是否需要在请求体中发送所有BeatInfo信息。这个lightBeatEnabled参数可以由服务端返回的心跳响应控制，默认情况是false，不会发送完整的BeatInfo信息。
    if (!lightBeatEnabled) {
        bodyMap.put("beat", JacksonUtils.toJson(beatInfo));
    }
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, beatInfo.getServiceName());
    params.put(CommonParams.CLUSTER_NAME, beatInfo.getCluster());
    params.put("ip", beatInfo.getIp());
    params.put("port", String.valueOf(beatInfo.getPort()));
    String result = reqApi(UtilAndComs.nacosUrlBase + "/instance/beat", params, bodyMap, HttpMethod.PUT);
    return JacksonUtils.toObj(result);
}
```

注意到**lightBeatEnabled参数用于控制是否需要在请求体中发送所有BeatInfo信息**。这个lightBeatEnabled参数可以由服务端返回的心跳响应控制，默认情况是false，不会发送完整的BeatInfo信息。

```java
// BeatReactor.BeatTask
// 2. 服务端可以决定客户端是否要发送所有BeatInfo信息
boolean lightBeatEnabled = false;
if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
  lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
}
```

#### 服务端找不到客户端实例，如何处理心跳

如果客户端向服务端发送心跳时，服务端没有在注册表中找到对应客户端实例，客户端会根据服务端响应状态码**RESOURCE_NOT_FOUND（20404）做特殊处理，会尝试向服务端发起一次注册请求**。

```java
// 3. 如果当前Instance在服务端没找到，尝试注册
//服务端找不到客户端实例，如何处理心跳
//如果客户端向服务端发送心跳时，服务端没有在注册表中找到对应客户端实例，客户端会根据服务端响应状态码RESOURCE_NOT_FOUND（20404）做特殊处理，会尝试向服务端发起一次注册请求。
if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
    Instance instance = new Instance();
    instance.setPort(beatInfo.getPort());
    instance.setIp(beatInfo.getIp());
    instance.setWeight(beatInfo.getWeight());
    instance.setMetadata(beatInfo.getMetadata());
    instance.setClusterName(beatInfo.getCluster());
    instance.setServiceName(beatInfo.getServiceName());
    instance.setInstanceId(instance.getInstanceId());
    instance.setEphemeral(true);
    try {

        serverProxy.registerService(beatInfo.getServiceName(),
                NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
    } catch (Exception ignore) {
    
    }
}
```

### 服务注册

NacosNamingService向Nacos服务端发送注册请求，直接调用NamingProxy的registerService方法。

注意入参**serviceName = Nacos服务名 = groupName + @@ + serviceName**。

```java
// NamingProxy
private final String namespaceId;
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

    final Map<String, String> params = new HashMap<String, String>(16);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.GROUP_NAME, groupName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("weight", String.valueOf(instance.getWeight()));
    params.put("enable", String.valueOf(instance.isEnabled()));
    params.put("healthy", String.valueOf(instance.isHealthy()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));
    params.put("metadata", JacksonUtils.toJson(instance.getMetadata()));
    // /nacos/v1/ns/instance
    reqApi(UtilAndComs.nacosUrlInstance, params, HttpMethod.POST);

}
```

没什么特殊逻辑，就是用Instance里的数据请求**POST /nacos/v1/ns/instance**。

## 2、服务注销

NacosNamingService的deregisterInstance方法负责注销服务，是服务注册反向操作，先取消心跳任务，再调用服务端注销。

```java
// NacosNamingService
 public void deregisterInstance(String serviceName, String groupName, Instance instance) throws NacosException {
     // 1. 取消心跳任务
     if (instance.isEphemeral()) {
         beatReactor.removeBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), instance.getIp(),
                                    instance.getPort());
     }
     // 2. 调用服务端注销 DELETE /nacos/v1/ns/instance。
     serverProxy.deregisterService(NamingUtils.getGroupedName(serviceName, groupName), instance);
 }
```



**BeatReactor**移除BeatInfo，设置BeatInfo的stopped属性为false，可以停止已经提交的BeatTask。

```java
// BeatReactor
// BeatInfo唯一标识 - BeatInfo
// BeatInfo的唯一标识=groupName + @@ + serviceName + # + ip + # + port
public final Map<String, BeatInfo> dom2Beat = new ConcurrentHashMap<String, BeatInfo>();
public void removeBeatInfo(String serviceName, String ip, int port) {
    BeatInfo beatInfo = dom2Beat.remove(buildKey(serviceName, ip, port));
    if (beatInfo == null) {
        return;
    }
    beatInfo.setStopped(true);
}
```

**NamingProxy**的**deregisterService**方法调用**DELETE /nacos/v1/ns/instance**。

```java
// NamingProxy
public void deregisterService(String serviceName, Instance instance) throws NacosException {
    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));

    reqApi(UtilAndComs.nacosUrlInstance, params, HttpMethod.DELETE);
}
```



## 3、服务发现

### 服务查询/订阅

服务查询以NacosNamingService的**getAllInstances**方法为例，获取所有服务对应实例。根据入参subscribe确定逻辑，当subscribe为true，表示需要订阅服务，会走**三层存储查询逻辑**；当subscribe为false，直接请求服务端获取实时注册表。这里只关心前者，因为前者包含了后者的逻辑。**所有查询逻辑都是由HostReactor处理**。

NacosNamingService#getAllInstances()

```java
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters,
        boolean subscribe) throws NacosException {

    ServiceInfo serviceInfo;

    /**
     * 实际上订阅和直接查询调用的都是GET /nacos/v1/ns/instance/list，区别在于订阅请求中的udpPort参数，带上了客户端的UDP端口号， 当查询的服务发生变化时，服务端会通过udp协议推送至客户端
     * 而直接查询请求，UDP端口号是0，这点在下一章服务端的时候会看到
     */
    if (subscribe) {
        // subscribe=true，走三层存储查询，订阅服务
        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
                StringUtils.join(clusters, ","));
    } else {
        // subscribe=false，直接通过NamingProxy获取服务端注册表
        serviceInfo = hostReactor
                .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),
                        StringUtils.join(clusters, ","));
    }
    List<Instance> list;
    if (serviceInfo == null || CollectionUtils.isEmpty(list = serviceInfo.getHosts())) {
        return new ArrayList<Instance>();
    }
    return list;
}
```



**HostReactor中有三个Map**：

```java
// HostReactor
// 服务更新任务
private final Map<String, ScheduledFuture<?>> futureMap = new HashMap<String, ScheduledFuture<?>>();
// 服务注册表
private final Map<String, ServiceInfo> serviceInfoMap;
// 服务更新表
private final Map<String, Object> updatingMap;
```

**serviceInfoMap**：服务唯一标识与服务信息ServiceInfo的映射关系。服务唯一标识=groupName+@@+serviceName+@@+clusterName，其中clusterName部分可以为空。

**updatingMap**：存储正在执行更新操作的service唯一标识。

**futureMap**：存储服务唯一标识与服务更新Future的映射关系。



![客户端注册表.png](../images/9b36ba8e56a64859ba4521fc01ce195c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)







HostReactor#getServiceInfo

```java
public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {

    NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
    String key = ServiceInfo.getKey(serviceName, clusters);
    // 1. 优先判断当前是否处于failover状态---由本地配置文件决定
    if (failoverReactor.isFailoverSwitch()) {
        return failoverReactor.getService(key);
    }
    // 2. 内存serviceInfoMap中查询ServiceInfo
    ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);

    if (null == serviceObj) {
        // 2-1. 内存中不存在，向服务端发起查询
        serviceObj = new ServiceInfo(serviceName, clusters);

        serviceInfoMap.put(serviceObj.getKey(), serviceObj);

        updatingMap.put(serviceName, new Object());
        //向服务端发起查询，并更新注册表
        updateServiceNow(serviceName, clusters);
        updatingMap.remove(serviceName);

    } else if (updatingMap.containsKey(serviceName)) {
        // 2-2. 内存中存在，如果正在更新注册表里的这个service，等待更新
        if (UPDATE_HOLD_INTERVAL > 0) {
            // hold a moment waiting for update finish
            synchronized (serviceObj) {
                try {
                    serviceObj.wait(UPDATE_HOLD_INTERVAL);
                } catch (InterruptedException e) {
                    NAMING_LOGGER
                            .error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                }
            }
        }
    }
    // 3. 提交一个任务，定时更新serviceInfo 当查询时futureMap中不存在对应service的更新任务Future，提交一个任务用于更新service对应注册表。
    scheduleUpdateIfAbsent(serviceName, clusters);
    // 4. 返回内存map中的serviceInfo
    return serviceInfoMap.get(serviceObj.getKey());
}
```

整个查询流程如上述代码。failover相关内容后续再看，注意几个关键点。

#### 内存注册表

**HostReactor**中存储了**服务注册表serviceInfoMap**。那么内存注册表的数据来源是哪里？

```java
// HostReactor
// 服务注册表
private final Map<String, ServiceInfo> serviceInfoMap;
```

**来源一：初始化更新**

对于某个**Service首次查询**，会调用NamingProxy实时查询服务端结果，更新HostReactor的内存注册表。

```java
// HostReactor.getServiceInfo
// 更新ServiceInfo
updatingMap.put(serviceName, new Object());
updateServiceNow(serviceName, clusters);
updatingMap.remove(serviceName);
```



立即更新逻辑如下，主要是调用**GET /nacos/v1/ns/instance/list**，然后**processServiceJson**更新内存中的注册表。

```java
// HostReactor
private void updateServiceNow(String serviceName, String clusters) {
    try {
        updateService(serviceName, clusters);
    } catch (NacosException e) {
        NAMING_LOGGER.error("[NA] failed to update serviceName: " + serviceName, e);
    }
}
public void updateService(String serviceName, String clusters) throws NacosException {
    // 1. 获取老的ServiceInfo
    ServiceInfo oldService = getServiceInfo0(serviceName, clusters);
    try {
        // 2. GET /nacos/v1/ns/instance/list    pushReceiver.getUdpPort() 携带了udpPort
        String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUdpPort(), false);
        // 3. 更新注册表
        if (StringUtils.isNotEmpty(result)) {
            processServiceJson(result);
        }
    } finally {
        if (oldService != null) {
            // 4. 通知等待在老serviceInfo上的线程（由于updatingMap中存在service而等待）
            synchronized (oldService) {
                oldService.notifyAll();
            }
        }
    }
}
```

**processServiceJson**的逻辑如下，更新注册表，发布InstancesChangeEvent事件，并将最新的ServiceInfo写入本地磁盘。

InstancesChangeNotifier处理InstancesChangeEvent事件

```java
public ServiceInfo processServiceJson(String json) {
    ServiceInfo serviceInfo = JacksonUtils.toObj(json, ServiceInfo.class);
    ServiceInfo oldService = serviceInfoMap.get(serviceInfo.getKey());

    if (pushEmptyProtection && !serviceInfo.validate()) {
      //empty or error push, just ignore
      return oldService;
    }

    boolean changed = false;
    // 如果已经存在ServiceInfo，比较新老ServiceInfo
    if (oldService != null) {

      if (oldService.getLastRefTime() > serviceInfo.getLastRefTime()) {
        NAMING_LOGGER.warn("out of date data received, old-t: " + oldService.getLastRefTime() + ", new-t: "
                           + serviceInfo.getLastRefTime());
      }

      //更新注册表
      serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);

      Map<String, Instance> oldHostMap = new HashMap<String, Instance>(oldService.getHosts().size());
      for (Instance host : oldService.getHosts()) {
        oldHostMap.put(host.toInetAddr(), host);
      }

      Map<String, Instance> newHostMap = new HashMap<String, Instance>(serviceInfo.getHosts().size());
      for (Instance host : serviceInfo.getHosts()) {
        newHostMap.put(host.toInetAddr(), host);
      }

      Set<Instance> modHosts = new HashSet<Instance>();
      Set<Instance> newHosts = new HashSet<Instance>();
      Set<Instance> remvHosts = new HashSet<Instance>();

      List<Map.Entry<String, Instance>> newServiceHosts = new ArrayList<Map.Entry<String, Instance>>(
        newHostMap.entrySet());
      for (Map.Entry<String, Instance> entry : newServiceHosts) {
        Instance host = entry.getValue();
        String key = entry.getKey();
        if (oldHostMap.containsKey(key) && !StringUtils
            .equals(host.toString(), oldHostMap.get(key).toString())) {
          modHosts.add(host);
          continue;
        }

        if (!oldHostMap.containsKey(key)) {
          newHosts.add(host);
        }
      }

      for (Map.Entry<String, Instance> entry : oldHostMap.entrySet()) {
        Instance host = entry.getValue();
        String key = entry.getKey();
        if (newHostMap.containsKey(key)) {
          continue;
        }

        if (!newHostMap.containsKey(key)) {
          remvHosts.add(host);
        }

      }

      if (newHosts.size() > 0) {
        changed = true;
        NAMING_LOGGER.info("new ips(" + newHosts.size() + ") service: " + serviceInfo.getKey() + " -> "
                           + JacksonUtils.toJson(newHosts));
      }

      if (remvHosts.size() > 0) {
        changed = true;
        NAMING_LOGGER.info("removed ips(" + remvHosts.size() + ") service: " + serviceInfo.getKey() + " -> "
                           + JacksonUtils.toJson(remvHosts));
      }

      if (modHosts.size() > 0) {
        changed = true;
        // 如果存在instance对应心跳任务，更新对应心跳任务的BeatInfo信息
        updateBeatInfo(modHosts);
        NAMING_LOGGER.info("modified ips(" + modHosts.size() + ") service: " + serviceInfo.getKey() + " -> "
                           + JacksonUtils.toJson(modHosts));
      }

      serviceInfo.setJsonFromServer(json);

      if (newHosts.size() > 0 || remvHosts.size() > 0 || modHosts.size() > 0) {
        // 发布InstancesChangeEvent事件，将serviceInfo写入本地磁盘用作failover
        NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                                                           serviceInfo.getClusters(), serviceInfo.getHosts()));
        DiskCache.write(serviceInfo, cacheDir);
      }

    } else {
      // 如果不存在老的ServiceInfo
      changed = true;
      NAMING_LOGGER.info("init new ips(" + serviceInfo.ipCount() + ") service: " + serviceInfo.getKey() + " -> "
                         + JacksonUtils.toJson(serviceInfo.getHosts()));
      // 更新注册表，发布InstancesChangeEvent事件，将serviceInfo写入本地磁盘用作failover
      //InstancesChangeNotifier.onEvent处理InstancesChangeEvent事件
      serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);
      NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                                                         serviceInfo.getClusters(), serviceInfo.getHosts()));
      serviceInfo.setJsonFromServer(json);
      DiskCache.write(serviceInfo, cacheDir);
    }

    MetricsMonitor.getServiceInfoMapSizeMonitor().set(serviceInfoMap.size());

    if (changed) {
      NAMING_LOGGER.info("current ips:(" + serviceInfo.ipCount() + ") service: " + serviceInfo.getKey() + " -> "
                         + JacksonUtils.toJson(serviceInfo.getHosts()));
    }

    return serviceInfo;
}
```



**来源二：定时更新**

当查询时**futureMap**中不存在对应service的更新任务Future，提交一个任务用于更新service对应注册表。

```java
// HostReactor.getServiceInfo
// 3. 提交一个任务，用于异步更新对应serviceInfo
scheduleUpdateIfAbsent(serviceName, clusters);
```



```java
// HostReactor
public void scheduleUpdateIfAbsent(String serviceName, String clusters) {
    if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
      return;
    }
    synchronized (futureMap) {
      if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
        return;
      }
      ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, clusters));
      futureMap.put(ServiceInfo.getKey(serviceName, clusters), future);
    }
}

public synchronized ScheduledFuture<?> addTask(UpdateTask task) {
  	return executor.schedule(task, DEFAULT_DELAY, TimeUnit.MILLISECONDS);
}
```

重点在于**UpdateTask**的逻辑。一旦用户执行了对于某个serivce的查询，这个UpdateTask任务会一直执行，服务端会控制这个定时任务的**时间间隔为10s**。

```java
public class UpdateTask implements Runnable {
    long lastRefTime = Long.MAX_VALUE;
    private final String clusters;
    private final String serviceName;
    private int failCount = 0;
    public void run() {
        // 下次拉取服务注册表的延迟时间，默认1s，服务端会控制到10s
        long delayTime = DEFAULT_DELAY;

        try {
            ServiceInfo serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));

            if (serviceObj == null) {
                // 从服务端获取实时注册表，更新至内存
                updateService(serviceName, clusters);
                return;
            }

            if (serviceObj.getLastRefTime() <= lastRefTime) {
                updateService(serviceName, clusters);
                serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
            } else {
                // if serviceName already updated by push, we should not override it
                // since the push data may be different from pull through force push
                refreshOnly(serviceName, clusters);
            }

            lastRefTime = serviceObj.getLastRefTime();

            if (!notifier.isSubscribed(serviceName, clusters) && !futureMap
                .containsKey(ServiceInfo.getKey(serviceName, clusters))) {
                // abort the update task
                NAMING_LOGGER.info("update task is stopped, service:" + serviceName + ", clusters:" + clusters);
                return;
            }
            if (CollectionUtils.isEmpty(serviceObj.getHosts())) {
                incFailCount();
                return;
            }
            // 使用服务端定义的延迟时间
            delayTime = serviceObj.getCacheMillis();
            resetFailCount();
        } catch (Throwable e) {
            incFailCount();
            NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
        } finally {
            // 提交延迟任务，执行下一次拉取服务注册表
            executor.schedule(this, Math.min(delayTime << failCount, DEFAULT_DELAY * 60), TimeUnit.MILLISECONDS);
        }
    }
}
```

**来源三：服务端推送**

**HostReactor构造**时，会创建一个**PushReceiver**。

```
public HostReactor(NamingProxy serverProxy, BeatReactor beatReactor, String cacheDir, boolean loadCacheAtStart,
        boolean pushEmptyProtection, int pollingThreadCount) {
    this.pushReceiver = new PushReceiver(this);
    // ...
}
```



**PushReceiver**负责处理服务端推送ServiceInfo信息，看到DatagramSocket知道服务端向客户端推送使用的是**UDP协议**。

```java
public class PushReceiver implements Runnable, Closeable {
    private ScheduledExecutorService executorService;
    private DatagramSocket udpSocket;
    private HostReactor hostReactor;
    private volatile boolean closed = false;
    public PushReceiver(HostReactor hostReactor) {
        try {
            this.hostReactor = hostReactor;
            this.udpSocket = new DatagramSocket();
            this.executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r);
                    thread.setDaemon(true);
                    thread.setName("com.alibaba.nacos.naming.push.receiver");
                    return thread;
                }
            });
            this.executorService.execute(this);
        } catch (Exception e) {
            NAMING_LOGGER.error("[NA] init udp socket failed", e);
        }
    }
}
```



PushReceiver实现Runnable接口的逻辑，发现还是调用HostReactor的processServiceJson方法解析报文并更新注册表。

```java
public void run() {
    while (!closed) {
        try {

            // byte[] is initialized with 0 full filled by default
            byte[] buffer = new byte[UDP_MSS];
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            // 等待服务端推送...
            udpSocket.receive(packet);

            String json = new String(IoUtils.tryDecompress(packet.getData()), UTF_8).trim();
            NAMING_LOGGER.info("received push data: " + json + " from " + packet.getAddress().toString());

            PushPacket pushPacket = JacksonUtils.toObj(json, PushPacket.class);
            String ack;
            if ("dom".equals(pushPacket.type) || "service".equals(pushPacket.type)) {
                // HostReactor处理报文，更新内存注册表
                hostReactor.processServiceJson(pushPacket.data);

                // send ack to server
                ack = "{\"type\": \"push-ack\"" + ", \"lastRefTime\":\"" + pushPacket.lastRefTime + "\", \"data\":"
                    + "\"\"}";
            } else if ("dump".equals(pushPacket.type)) {
                // dump data to server
                ack = "{\"type\": \"dump-ack\"" + ", \"lastRefTime\": \"" + pushPacket.lastRefTime + "\", \"data\":"
                    + "\"" + StringUtils.escapeJavaScript(JacksonUtils.toJson(hostReactor.getServiceInfoMap()))
                    + "\"}";
            } else {
                // do nothing send ack only
                ack = "{\"type\": \"unknown-ack\"" + ", \"lastRefTime\":\"" + pushPacket.lastRefTime
                    + "\", \"data\":" + "\"\"}";
            }
            // 发送ack报文给服务端
            udpSocket.send(new DatagramPacket(ack.getBytes(UTF_8), ack.getBytes(UTF_8).length,
                                              packet.getSocketAddress()));
        } catch (Exception e) {
            if (closed) {
                return;
            }
            NAMING_LOGGER.error("[NA] error while receiving push data", e);
        }
    }
}
```

**总结一下客户端注册表的更新，主要分为推和拉两种方式**：

- **拉**：通过查询服务触发注册表更新，每个服务会对应一个UpdateTask每隔10s更新服务对应的注册表
- **推**：通过UDP协议，服务端向客户端推送注册表信息

![客户端注册表更新.png](../images/1889d75083844ffab1fa7ad3138c5f1f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)





#### 故障转移

![failover.png](../images/6449bacaddd043a58fbde53dba975479~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



**FailoverReactor**负责客户端服务发现的故障转移。如果客户端与服务端因为各种原因无法正常获取到注册表，Nacos**允许客户端使用本地磁盘中的注册表加载至内存使用，前提是用户需要修改磁盘上的failover开关**。

无论是加载开关还是落盘failover注册表，都是通过定时任务实现的，任务在FailoverReactor构造时提交。

```java
public class FailoverReactor implements Closeable {
		// 本地缓存路径
    private final String failoverDir;
    private final HostReactor hostReactor;
    private final ScheduledExecutorService executorService;
    // failover内存注册表
 		// key=cluster+group+service
    private Map<String, ServiceInfo> serviceMap = new ConcurrentHashMap<String, ServiceInfo>();
    public FailoverReactor(HostReactor hostReactor, String cacheDir) {
        this.hostReactor = hostReactor;
        this.failoverDir = cacheDir + "/failover";
        this.executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setDaemon(true);
                thread.setName("com.alibaba.nacos.naming.failover");
                return thread;
            }
        });
        // 初始化方法
        this.init();
    }
}
```



**故障转移开关**

**SwitchRefresher**负责处理刷新开关逻辑。

FailoverReactor#init

```java
public void init() {
    // 根据本地文件配置，刷新fail-over开关
    executorService.scheduleWithFixedDelay(new SwitchRefresher(), 0L, 5000L, TimeUnit.MILLISECONDS);
}
```

SwitchRefresher根据cacheDir路径下00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00文件内容判断是否开启failover，文件内容1表示开启0表示关闭。如果开启，**FailoverFileReader**会加载磁盘中的注册表至内存serviceMap。

```java
class SwitchRefresher implements Runnable {

    long lastModifiedMillis = 0L;

    @Override
    public void run() {
        try {

            //SwitchRefresher根据cacheDir路径下00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00文件内容判断是否开启failover，文件内容1表示开启0表示关闭
            File switchFile = new File(failoverDir + UtilAndComs.FAILOVER_SWITCH);
            if (!switchFile.exists()) {
                //文件不存在 默认关闭
                switchParams.put("failover-mode", "false");
                NAMING_LOGGER.debug("failover switch is not found, " + switchFile.getName());
                return;
            }

            long modified = switchFile.lastModified();

            if (lastModifiedMillis < modified) {
                lastModifiedMillis = modified;
                String failover = ConcurrentDiskUtil.getFileContent(failoverDir + UtilAndComs.FAILOVER_SWITCH,
                        Charset.defaultCharset().toString());
                if (!StringUtils.isEmpty(failover)) {
                    String[] lines = failover.split(DiskCache.getLineSeparator());

                    for (String line : lines) {
                        String line1 = line.trim();
                        if ("1".equals(line1)) {
                            switchParams.put("failover-mode", "true");
                            NAMING_LOGGER.info("failover-mode is on");
                            //如果开启，FailoverFileReader会加载磁盘中的注册表至内存serviceMap
                            new FailoverFileReader().run();
                        } else if ("0".equals(line1)) {
                            switchParams.put("failover-mode", "false");
                            NAMING_LOGGER.info("failover-mode is off");
                        }
                    }
                } else {
                    switchParams.put("failover-mode", "false");
                }
            }

        } catch (Throwable e) {
            NAMING_LOGGER.error("[NA] failed to read failover switch.", e);
        }
    }
}
```



**内存注册表落盘**

**DiskFileWriter负责将内存注册表落盘**。

```java
public void init() {
    // ...
    // 每天执行一次，将内存中的服务注册表写入本地磁盘
    executorService.scheduleWithFixedDelay(new DiskFileWriter(), 30, DAY_PERIOD_MINUTES, TimeUnit.MINUTES);

    // 10s后，将内存中的服务注册表，写入本地磁盘
    executorService.schedule(new Runnable() {
        @Override
        public void run() {
           // ...
           new DiskFileWriter().run();
           // ...
        }
    }, 10000L, TimeUnit.MILLISECONDS);
}
```

**DiskFileWriter使用HostReactor中的serviceInfoMap落盘**。

```java
class DiskFileWriter extends TimerTask {
    @Override
    public void run() {
        Map<String, ServiceInfo> map = hostReactor.getServiceInfoMap();
        for (Map.Entry<String, ServiceInfo> entry : map.entrySet()) {
            ServiceInfo serviceInfo = entry.getValue();
            // ... 省略部分过滤逻辑
            DiskCache.write(serviceInfo, failoverDir);
        }
    }
}
```

#### 订阅 or 查询

用户代码调用 NacosNamingService的getAllInstances方法，如果subscribe=true，走订阅逻辑；subscribe=false，走查询逻辑。

```java
// NacosNamingService
@Override
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters,
        boolean subscribe) throws NacosException {
    if (subscribe) {
        // subscribe=true，走三层存储查询，订阅服务
    } else {
        // subscribe=false，直接通过NamingProxy获取服务端注册表
    }
}
```

**实际上订阅和查询调用的都是GET /nacos/v1/ns/instance/list**，区别在于订阅请求中的**udpPort参数**，带上了客户端的UDP端口号，而查询请求，UDP端口号是0，这点在下一章服务端的时候会看到。

```java
public String queryList(String serviceName, String clusters, int udpPort, boolean healthyOnly)
        throws NacosException {
    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put("clusters", clusters);
    params.put("udpPort", String.valueOf(udpPort));
    params.put("clientIP", NetUtils.localIP());
    params.put("healthyOnly", String.valueOf(healthyOnly));
    return reqApi(UtilAndComs.nacosUrlBase + "/instance/list", params, HttpMethod.GET);
}
```



### 服务监听

服务订阅是为了更新客户端的内存注册表。

除此之外，用户代码可以使用**NacosNamingService的subscribe**方法监听服务注册表变更，实现自己的业务逻辑。

```java
// Example
nacosNamingService.subscribe("nacos.test.3", new AbstractEventListener() {
    @Override
    public Executor getExecutor() {
        return executor;
    }
    @Override
    public void onEvent(Event event) {
        System.out.println(((NamingEvent) event).getServiceName());
        System.out.println(((NamingEvent) event).getInstances());
    }
});
```

NacosNamingService内部**委托给HostReactor**处理。

```java
// NacosNamingService
@Override
public void subscribe(String serviceName, String groupName, List<String> clusters, EventListener listener)
        throws NacosException {
    hostReactor.subscribe(NamingUtils.getGroupedName(serviceName, groupName), StringUtils.join(clusters, ","),
            listener);
}
```

HostReactor先调用**InstancesChangeNotifier**注册监听器，然后执行了一次getServiceInfo查询方法（上面讲了），确保初始化注册表信息并开启定时任务拉取注册表。

```java
// HostReactor
public void subscribe(String serviceName, String clusters, EventListener eventListener) {
    // 1. InstancesChangeNotifier注册监听器
    notifier.registerListener(serviceName, clusters, eventListener);
    // 2. 查询ServiceInfo
    getServiceInfo(serviceName, clusters);
}
```





#### InstancesChangeNotifier

**InstancesChangeNotifier负责注册监听器和回调监听器**。

**registerListener**方法将service标识和对应监听器存储到map中

```java
public class InstancesChangeNotifier extends Subscriber<InstancesChangeEvent> {
    // 监听注册表
    // service唯一标识groupName+@@+serviceName+@@+clusterName - 监听器
    private final Map<String, ConcurrentHashSet<EventListener>> listenerMap = new ConcurrentHashMap<String, ConcurrentHashSet<EventListener>>();

    private final Object lock = new Object();

    public void registerListener(String serviceName, String clusters, EventListener listener) {
        String key = ServiceInfo.getKey(serviceName, clusters);
        ConcurrentHashSet<EventListener> eventListeners = listenerMap.get(key);
        if (eventListeners == null) {
            synchronized (lock) {
                eventListeners = listenerMap.get(key);
                if (eventListeners == null) {
                    eventListeners = new ConcurrentHashSet<EventListener>();
                    listenerMap.put(key, eventListeners);
                }
            }
        }
        eventListeners.add(listener);
    }
}
```

此外，InstancesChangeNotifier实现Subscriber接口，负责处理InstancesChangeEvent服务实例变更事件。

```java
// InstancesChangeNotifier

/**
     *  HostReactor#processServiceJson()发布InstancesChangeEvent事件
     * 处理InstancesChangeEvent服务实例变更事件。
     * @param event {@link Event}
 */
public void onEvent(InstancesChangeEvent event) {
    String key = ServiceInfo.getKey(event.getServiceName(), event.getClusters());
    //获取对该服务感兴趣的监听器  如何添加对某个服务的监听？ NacosNamingService.subscribe()
    ConcurrentHashSet<EventListener> eventListeners = listenerMap.get(key);
    if (CollectionUtils.isEmpty(eventListeners)) {
      return;
    }
    for (final EventListener listener : eventListeners) {
      final com.alibaba.nacos.api.naming.listener.Event namingEvent = transferToNamingEvent(event);
      if (listener instanceof AbstractEventListener && ((AbstractEventListener) listener).getExecutor() != null) {
        ((AbstractEventListener) listener).getExecutor().execute(new Runnable() {
          @Override
          public void run() {
            //触发事件
            listener.onEvent(namingEvent);
          }
        });
        continue;
      }
      listener.onEvent(namingEvent);
    }
}
```



在上述内存注册表更新时，会触发服务实例变更事件。

**HostReactor#processServiceJson**更新注册表。

```java
// HostReactor
public ServiceInfo processServiceJson(String json) {
    ServiceInfo serviceInfo = JacksonUtils.toObj(json, ServiceInfo.class);
    ServiceInfo oldService = serviceInfoMap.get(serviceInfo.getKey());
    boolean changed = false;
    // 如果已经存在ServiceInfo，比较新老ServiceInfo
    if (oldService != null) {
        // ...
        // 发布InstancesChangeEvent事件，将serviceInfo写入本地磁盘用作failover
        if (newHosts.size() > 0 || remvHosts.size() > 0 || modHosts.size() > 0) {
            NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                    serviceInfo.getClusters(), serviceInfo.getHosts()));
            DiskCache.write(serviceInfo, cacheDir);
        }
    } else {
        // 如果不存在ServiceInfo
        changed = true;
        // 更新注册表，发布InstancesChangeEvent事件，将serviceInfo写入本地磁盘用作failover
        serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);
        NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                serviceInfo.getClusters(), serviceInfo.getHosts()));
        serviceInfo.setJsonFromServer(json);
        DiskCache.write(serviceInfo, cacheDir);
    }
    return serviceInfo;
}
```



# 总结

- 服务注册

  对于临时实例（默认），Instance.ephemeral=true，客户端将自身Instance实例信息通过POST /nacos/v1/ns/instance注册到服务端。

  此后，客户端默认每隔5s（instance的元数据preserved.heart.beat.interval）向服务端发起心跳请求PUT /nacos/v1/ns/instance/beat。

  如果发送心跳时，服务端返回RESOURCE_NOT_FOUND（20404）异常，表示Instance还未执行注册，这时候由客户端在心跳过程中，主动向服务端发起一次注册请求。

- 服务注销

  服务注册的相反操作，首先会取消定时心跳任务，然后调用服务端DELETE /nacos/v1/ns/instance将当前实例从服务列表中注销。

- 服务发现

  对于**服务查询**，用户即可以走订阅逻辑，也可以走实时查询逻辑，具体取决于NacosNamingService的getAllInstances方法的第四个入参，subscribe=true表示走服务订阅流程，subscribe=false代表不走服务订阅流程，直接查询服务端最新服务注册表，前者包含了后者的所有逻辑。

```java
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters, boolean subscribe) 
```

**服务订阅流程**如下：

1. failover故障转移开关开启的情况下，会读取本地文件系统中的注册表，加载到FailoverReactor的serviceMap变量中。用户查询时，会读取FailoverReactor的serviceMap获取服务信息。
2. 一般情况下，failover开关不开启，这里会**优先读取HostReactor.serviceMap内存注册表**。
3. 如果HostReactor.serviceMap中没有读到服务，会请求Nacos服务端。一方面获取服务注册信息并更新HostReactor.serviceMap内存注册表，另一方面查询请求**将本地启动的UDP端口告知服务端**，告知服务端自己订阅服务。
4. 对于每个订阅服务，客户端会确保开启**服务更新任务UpdateTask**，定时请求服务端获取最新注册表。客户端默认1秒拉取一次，但是服务端会通过返回报文控制客户端**10秒拉取一次**。

**内存注册表会在三个场景下更新**：

1. 首次服务查询时，调用GET /nacos/v1/ns/instance/list获取实时注册表，更新到本地内存注册表中
2. 查询服务会触发注册表更新，每个服务会对应一个UpdateTask每隔10s调用GET /nacos/v1/ns/instance/list获取实时注册表，更新服务对应的注册表
3. 通过UDP协议，服务端向客户端推送注册表信息

感知服务变更：当内存注册表发生变更时，会发布InstancesChangeEvent事件。InstancesChangeNotifier处理这个事件，将变更信息通知到所有通过NamingService的subscribe方法注册监听的客户端代码。













































