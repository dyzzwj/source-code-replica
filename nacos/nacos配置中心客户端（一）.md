​																		nacos配置中心客户端										            		

本章分析nacos1.4.1版本的配置中心客户端。

- 如何使用
- Nacos配置中心模型（namespace、group、dataId）
- Nacos配置客户端ConfigService的解析，包括配置查询和配置监听

# 一、使用案例

```java
public class MyConfigExample {
    public static void main(String[] args) throws NacosException {
        String serverAddr = "localhost";
        String dataId = "cfg0"; // dataId
        String group = "DEFAULT_GROUP"; // group
        Properties properties = new Properties();
        // nacos-server地址
        properties.put("serverAddr", serverAddr);
        // namespace/tenant
        properties.put("namespace", "dca8ec01-bca3-4df9-89ef-8ab299a37f73"); 
        ConfigService configService = NacosFactory.createConfigService(properties);
        // 1. 注册监听
        configService.addListener(dataId, group, new AbstractListener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                System.out.println("receive config info ：" + configInfo);
            }
        });
        // 2. 查询初始配置
        String config = configService.getConfig(dataId, group, 3000);
        System.out.println("init config : " + config);
        // 3. 修改配置
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String next = scanner.next();
            if ("exit".equals(next)) {
                break;
            }
            configService.publishConfig(dataId, group, next);
        }
    }
}
```



# 二、配置中心模型



![模型.png](../images/4508dc4aa30849459c197c2a6f825e68~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215623264.awebp)

**Namspace（Tenant）：命名空间（租户），**默认命名空间是public。一个命名空间可以包含多个Group，在Nacos源码里有些变量是tenant租户，和命名空间是一个东西。

![控制台-命名空间.png](../images/3b896ccf70e74990871fc2252425aaf9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215637218.awebp)

**Group：组**，默认分组是DEFAULT_GROUP。一个组可以包含多个dataId。

**DataId：译为数据id，在nacos中DataId代表一整个配置文件，是配置的最小单位**。和apollo不同，apollo的最小单位是一个配置项key。

![控制台-ns和group和dataId.png](../images/efa49926662f41ca9d2f834cd528de70~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215647960.awebp)



# 三、ConfigService

![ConfigService.png](../images/865b84c0b90c42ecb1d9452dc0794963~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215717070.awebp)

**ConfigService**是Nacos暴露给客户端的配置服务接口，一个Nacos配置中心+一个Namespace=一个ConfigService实例。

```java
Properties properties = new Properties();
properties.put("serverAddr", serverAddr);
properties.put("namespace", "dca8ec01-bca3-4df9-89ef-8ab299a37f73");
ConfigService configService = NacosFactory.createConfigService(properties);
```



````java
public class NacosFactory {
    /**
     * Create config service.
     */
    public static ConfigService createConfigService(Properties properties) throws NacosException {
        return ConfigFactory.createConfigService(properties);
    }
}
public class ConfigFactory {
    /**
     * Create Config.
     */
    public static ConfigService createConfigService(Properties properties) throws NacosException {
      Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
      Constructor constructor = driverImplClass.getConstructor(Properties.class);
      ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
      return vendorImpl;
    }
}
````

为什么要通过反射创建NacosConfigService实现类？主要是为了将api层单独拆分出来。

ConfigService主要功能包括：

- 配置增删改查
- 配置监听注册



```java
public interface ConfigService {
    // 配置增删改查
    String getConfig(String dataId, String group, long timeoutMs) throws NacosException;
    String getConfigAndSignListener(String dataId, String group, long timeoutMs, Listener listener) throws NacosException;
    boolean publishConfig(String dataId, String group, String content) throws NacosException;
    boolean publishConfig(String dataId, String group, String content, String type) throws NacosException;
    boolean removeConfig(String dataId, String group) throws NacosException;
  	// 注册监听
    void addListener(String dataId, String group, Listener listener) throws NacosException;
    void removeListener(String dataId, String group, Listener listener);
    // NacosConfigServer状态 UP/DOWN
    String getServerStatus();
    // 资源关闭
    void shutDown() throws NacosException;
}
```





## 1、配置查询

配置的增删改查，都是由NacosConfigService实现的，重点看配置查询。

![配置查询.png](../images/4666e153bcae4611bbe5482b52766050~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215729210.awebp)

Nacos客户端获取配置的入口方法是**NacosConfigService#getConfigInner**。

```java
private final ClientWorker worker;
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = null2defaultGroup(group); // group默认设置为DEFAULT_GROUP
    ConfigResponse cr = new ConfigResponse();
    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);

    // LEVEL1 : 使用本地文件系统的failover配置
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        cr.setContent(content);
        content = cr.getContent();
        return content;
    }

    // LEVEL2 : 读取config-server实时配置，并将snapshot保存到本地文件系统
    try {
        String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        cr.setContent(ct[0]);
        content = cr.getContent();
        return content;
    } catch (NacosException ioe) {
        if (NacosException.NO_RIGHT == ioe.getErrCode()) {
            throw ioe;
        }
        // 非403错误进入LEVEL3
        LOGGER.warn(...);
    }

    // LEVEL3 : 如果读取config-server发生非403Forbidden错误，使用本地snapshot
    content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
    cr.setContent(content);
    content = cr.getContent();
    return content;
}
```



**failover文件**在Nacos里是优先级最高的，如果failover文件存在则不会使用nacos服务端的配置，永远会使用failover文件，即使服务端的配置发生了变化，类似于Apollo中-Denv=LOCAL时只会使用本地配置文件。需要注意的是，**Nacos的failover文件内容没有更新的入口，也就是说这个文件只能在文件系统中修改生效，生效时机在长轮询过程中**。

failover文件的路径是，这里agentName拼接逻辑比较复杂就不看了：

- 默认namespace：/{user.home}/{agentName}_nacos/data/config-data/{group}/{dataId}
- 指定namespace：/{user.home}/{agentName}_nacos/data/config-data-tenant/{namespace}/{group}/{dataId}

```java
// LocalConfigInfoProcessor
static File getFailoverFile(String serverName, String dataId, String group, String tenant) {
    File tmp = new File(LOCAL_SNAPSHOT_PATH, serverName + "_nacos");
    tmp = new File(tmp, "data");
    if (StringUtils.isBlank(tenant)) {
        tmp = new File(tmp, "config-data");
    } else {
        tmp = new File(tmp, "config-data-tenant");
        tmp = new File(tmp, tenant);
    }
    return new File(new File(tmp, group), dataId);
}
```

一般情况下，failover文件不会存在，那么都会走**ClientWorker.getServerConfig**方法。这个方法一方面是查询nacos服务端的最新配置，另一方面是更新snapshot文件。

```java
// ClientWorker
public String[] getServerConfig(String dataId, String group, String tenant, long readTimeout) throws NacosException {
    String[] ct = new String[2];
    if (StringUtils.isBlank(group)) {
        group = Constants.DEFAULT_GROUP;
    }

    HttpRestResult<String> result = null;
    try {
        Map<String, String> params = new HashMap<String, String>(3);
        if (StringUtils.isBlank(tenant)) {
            params.put("dataId", dataId);
            params.put("group", group);
        } else {
            params.put("dataId", dataId);
            params.put("group", group);
            params.put("tenant", tenant);
        }
        // 1. 请求/v1/cs/configs
        result = agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout);
    } catch (Exception ex) {
        throw new NacosException(NacosException.SERVER_ERROR, ex);
    }
    // 2. 处理返回结果，如果200和404，更新本地snapshot文件
    switch (result.getCode()) {
        case HttpURLConnection.HTTP_OK:
            LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.getData());
            ct[0] = result.getData();
            if (result.getHeader().getValue(CONFIG_TYPE) != null) {
                ct[1] = result.getHeader().getValue(CONFIG_TYPE);
            } else {
                ct[1] = ConfigType.TEXT.getType();
            }
            return ct;
        case HttpURLConnection.HTTP_NOT_FOUND:
            LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, null);
            return ct;
        // ... 省略其他状态，都是抛出NacosException
    }
}
```





如果ClientWorker.getServerConfig执行失败，且非403错误，会读取**snapshot文件兜底**。

snapshot文件的路径：

- 默认namespace：/{user.home}/{agentName}_nacos/snapshot/{group}/{dataId}
- 指定namespace：/{user.home}/{agentName}_nacos/snapshot-tenant/{namespace}/{group}/{dataId}



```java
static File getSnapshotFile(String envName, String dataId, String group, String tenant) {
    File tmp = new File(LOCAL_SNAPSHOT_PATH, envName + "_nacos");
    if (StringUtils.isBlank(tenant)) {
        tmp = new File(tmp, "snapshot");
    } else {
        tmp = new File(tmp, "snapshot-tenant");
        tmp = new File(tmp, tenant);
    }
    return new File(new File(tmp, group), dataId);
}
```

**综上所述，ConfigService获取配置，是不会走内存缓存的，要么是读取文件系统中的文件，要么是查询nacos-server的实时配置**。

## 2、配置监听

![ConfigService.png](../images/47a1691f17b44de498f23912d6b20e4c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215749134.awebp)

### 核心类

#### ClientWorker

**ClientWorker**主要的任务就是执行长轮询。

```java
public class ClientWorker implements Closeable {
	  // 检测是否需要提交longPolling任务到executorService，如果需要则提交
    final ScheduledExecutorService executor;
    // 执行长轮询，一般情况下执行listener回调也是在这个线程里
    final ScheduledExecutorService executorService;
    // groupKey -> cacheData
    private final ConcurrentHashMap<String, CacheData> cacheMap = new ConcurrentHashMap<String, CacheData>();
    // 认为是个httpClient
    private final HttpAgent agent;
    // 钩子管理器
    private final ConfigFilterChainManager configFilterChainManager;
    // nacos服务端是否健康
    private boolean isHealthServer = true;
    // 长轮询超时时间 默认30s
    private long timeout;
    // 当前长轮询任务数量
    private double currentLongingTaskCount = 0;
    // 长轮询发生异常，默认延迟2s进行下次长轮询
    private int taskPenaltyTime;
    // 是否在添加监听器时，主动获取最新配置
    private boolean enableRemoteSyncConfig = false;
}
```

**groupKey**：groupKey是为了确定一个ConfigService下唯一一个配置文件，**groupKey=dataId+group{+namespace}**。

**cacheMap**：存储groupKey和配置文件CacheData的映射关系。

**agent**：普通的HttpClient。



ClientWorker构造时会创建两个执行器。

- **executor**：负责检测当前情况（cacheMap大小及当前已经提交的长轮询任务数），是否需要提交新的长轮询任务到executorService中，固定线程数=1。
- **executorService**：负责执行长轮询任务，固定线程数=核数。

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,
        final Properties properties) {
    this.agent = agent;
    this.configFilterChainManager = configFilterChainManager;
    // 初始化一些参数，如：timeout
    init(properties);
		// 单线程执行器
    this.executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });
    // 执行LongPollingRunnable的执行器，固定线程数=核数
    this.executorService = Executors
            .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r);
                    t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                    t.setDaemon(true);
                    return t;
                }
            });
    // 检测并提交LongPollingRunnable到this.executorService
    this.executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}
```

#### CacheData

**CacheData是配置文件的抽象，一个groupKey对应一个CacheData**。CacheData上会注册监听器，当CacheData的配置 发生变化时，会触发监听器。



```java
// agentName
private final String name;
// dataId
public final String dataId;
// group
public final String group;
// namespace
public final String tenant;
// 注册在这个配置上的监听器
private final CopyOnWriteArrayList<ManagerListenerWrap> listeners;
// 配置的md5
private volatile String md5;
// 是否使用failover配置文件
private volatile boolean isUseLocalConfig = false;
// failover配置文件的上次更新时间戳
private volatile long localConfigLastModified;
// 配置
private volatile String content;
// 所属长轮询任务id
private int taskId;
// 是否正在初始化
private volatile boolean isInitializing = true;
// 配置文件类型 如：TEXT、JSON、YAML
private String type;
// 对查询配置的请求和响应提供钩子处理
private final ConfigFilterChainManager configFilterChainManager;
```



CacheData在构造时会加载本地文件系统的配置内容到conent里并计算其md5。

```java
public CacheData(ConfigFilterChainManager configFilterChainManager, String name, String dataId, String group,
        String tenant) {
    if (null == dataId || null == group) {
        throw new IllegalArgumentException("dataId=" + dataId + ", group=" + group);
    }
    this.name = name;
    this.configFilterChainManager = configFilterChainManager;
    this.dataId = dataId;
    this.group = group;
    this.tenant = tenant;
    listeners = new CopyOnWriteArrayList<ManagerListenerWrap>();
    this.isInitializing = true;
    // 这里会从本地文件系统加载配置内容，failover > snapshot
    this.content = loadCacheContentFromDiskLocal(name, dataId, group, tenant);
    this.md5 = getMd5String(content);
}
```



### 注册监听

注册监听器的案例如下。

```java
configService.addListener(dataId, group, new AbstractListener() {
    @Override
    public void receiveConfigInfo(String configInfo) {
        System.out.println("receive config info ：" + configInfo);
    }
});
```



NacosConfigService委托**ClientWorker**将监听器注册到CacheData中。

```java
// ClientWorker
public void addTenantListeners(String dataId, String group, List<? extends Listener> listeners)
        throws NacosException {
    group = null2defaultGroup(group);
    String tenant = agent.getTenant();
    // 获取CacheData
    CacheData cache = addCacheDataIfAbsent(dataId, group, tenant);
    // 给CacheData注册监听器
    for (Listener listener : listeners) {
        cache.addListener(listener);
    }
}
```



首先，如果当前groupKey对应的CacheData不存在，将会创建；否则会直接返回对应CacheData。

```java
public CacheData addCacheDataIfAbsent(String dataId, String group, String tenant) throws NacosException {
    String key = GroupKey.getKeyTenant(dataId, group, tenant);
    // 1 如果缓存中已经存在，直接返回
    CacheData cacheData = cacheMap.get(key);
    if (cacheData != null) {
        return cacheData;
    }
    // 2 创建CacheData，这里会使用本地配置文件设置为初始配置
    cacheData = new CacheData(configFilterChainManager, agent.getName(), dataId, group, tenant);
    // 3 多线程操作cacheMap再次校验是否已经缓存了cacheData
    CacheData lastCacheData = cacheMap.putIfAbsent(key, cacheData);
    // 4 如果当前线程成功设置了key-cacheData，返回cacheData
    if (lastCacheData == null) {
        if (enableRemoteSyncConfig) { // 是否允许添加监听时实时同步配置，默认false
            String[] ct = getServerConfig(dataId, group, tenant, 3000L);
            cacheData.setContent(ct[0]);
        }
        // 计算所属长轮询任务id
        int taskId = cacheMap.size() / (int) ParamUtil.getPerTaskConfigSize();
        cacheData.setTaskId(taskId);
        lastCacheData = cacheData;
    }
  	// 这里设置cacheData正在初始化，让下次长轮询立即返回结果
    lastCacheData.setInitializing(true);
    // 5 否则返回的cacheData是老的cacheData
    return lastCacheData;
}
```

确保CacheData创建以后，注册Listener到CacheData上。

```java
// CacheData
// 注册在这个tenant-group-dataId配置上的监听器
private final CopyOnWriteArrayList<ManagerListenerWrap> listeners;
public void addListener(Listener listener) {
    ManagerListenerWrap wrap =
            (listener instanceof AbstractConfigChangeListener) ? new ManagerListenerWrap(listener, md5, content)
                    : new ManagerListenerWrap(listener, md5);

    if (listeners.addIfAbsent(wrap)) {
        LOGGER.info("[{}] [add-listener] ok, tenant={}, dataId={}, group={}, cnt={}", name, tenant, dataId, group,
                listeners.size());
    }
}
```



### 长轮询

#### 长轮询的开启时机

构造ClientWorker时，会开启一个定时任务，**每隔10ms执行一次checkConfigInfo方法**。

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,
        final Properties properties) {
     // ...
    // 检测并提交LongPollingRunnable到this.executorService
    this.executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}
```



checkConfigInfo判断当前CacheData数量，是否要开启一个长轮询任务。判断依据是，**当前长轮询任务数量 < Math.ceil(cacheMap大小 / 3000)，则开启一个新的长轮询任务**。配置文件不多的情况下，最多也就一个长轮询任务。

```java
// ClientWorker
public void checkConfigInfo() {
    // cacheMap大小
    int listenerSize = cacheMap.size();
    // cacheMap大小 / 3000 向上取整
    int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
    // 计算longingTaskCount 大于 当前实际长轮询任务数量
    if (longingTaskCount > currentLongingTaskCount) {
        for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
            // 开启新的长轮询任务
            executorService.execute(new LongPollingRunnable(i));
        }
        currentLongingTaskCount = longingTaskCount;
    }
}
```





**所以开启长轮询任务的时机，一般是注册监听之后创建了CacheData，checkConfigInfo定时任务扫描到需要开启新的长轮询任务时，触发长轮询任务提交**。

#### 长轮询整体流程

**LongPollingRunnable长轮询任务**的主要流程如下。

```java
class LongPollingRunnable implements Runnable {
    private final int taskId;
    public LongPollingRunnable(int taskId) {
        this.taskId = taskId;
    }
    @Override
    public void run() {
        // 当前长轮询任务负责的CacheData集合
        List<CacheData> cacheDatas = new ArrayList<CacheData>();
        // 正在初始化的CacheData 即刚构建的CacheData，内部的content仍然是snapshot版本
        List<String> inInitializingCacheList = new ArrayList<String>();
        try {
            // 1. 对于failover配置文件的处理
            for (CacheData cacheData : cacheMap.values()) {
                if (cacheData.getTaskId() == taskId) {
                    cacheDatas.add(cacheData);
                    try {
                        // 判断cacheData是否需要使用failover配置，设置isUseLocalConfigInfo
                        // 如果需要则更新内存中的配置
                        checkLocalConfig(cacheData);
                        // 使用failover配置则检测content内容是否发生变化，如果变化则通知监听器
                        if (cacheData.isUseLocalConfigInfo()) {
                            cacheData.checkListenerMd5();
                        }
                    } catch (Exception e) {
                        LOGGER.error("get local config info error", e);
                    }
                }
            }

            // 2. 对于所有非failover配置，执行长轮询，返回发生改变的groupKey
            List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);

            for (String groupKey : changedGroupKeys) {
                String[] key = GroupKey.parseKey(groupKey);
                String dataId = key[0];
                String group = key[1];
                String tenant = null;
                if (key.length == 3) {
                    tenant = key[2];
                }
                try {
                    // 3. 对于发生改变的配置，查询实时配置并保存snapshot
                    String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                    // 4. 更新内存中的配置
                    CacheData cache = cacheMap.get(GroupKey.getKeyTenant(dataId, group, tenant));
                    cache.setContent(ct[0]);
                    if (null != ct[1]) {
                        cache.setType(ct[1]);
                    }
                } catch (NacosException ioe) {
                    LOGGER.error(message, ioe);
                }
            }
            // 5. 对于非failover配置，触发监听器
            for (CacheData cacheData : cacheDatas) {
                // 排除failover文件
                if (!cacheData.isInitializing() || inInitializingCacheList
                        .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                    // 校验md5是否发生变化，如果发生变化通知listener
                    cacheData.checkListenerMd5();
                    cacheData.setInitializing(false);
                }
            }
            inInitializingCacheList.clear();
            // 6-1. 都执行完成以后，再次提交长轮询任务
            executorService.execute(this);
        } catch (Throwable e) {
            // 6-2. 如果长轮询执行发生异常，延迟2s执行下一次长轮询
            executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
        }
    }
}
```

长轮询任务分为几个步骤：

- **处理failover配置**：判断当前CacheData是否使用failover配置（ClientWorker.checkLocalConfig），如果使用failover配置，则校验本地配置文件内容是否发生变化，发生变化则触发监听器（CacheData.checkListenerMd5）。这一步其实和长轮询无关。
- **对于所有非failover配置，执行长轮询**，返回发生改变的groupKey（ClientWorker.checkUpdateDataIds）。
- 根据返回的groupKey，**查询服务端实时配置并保存snapshot**（ClientWorker.getServerConfig）
- **更新内存CacheData的配置content**。
- **校验配置是否发生变更，通知监听器**（CacheData.checkListenerMd5）。
- **如果正常执行本次长轮询，立即提交长轮询任务，执行下一次长轮询；发生异常，延迟2s提交长轮询任务**。

![客户端长轮询步骤.png](../images/b25531b9b0164c3295c5734f6fd44085~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215821811.awebp)



#### 什么情况下会使用failover本地配置

长轮询任务不仅仅向服务端发起请求获取配置发生变更的groupKey，而且执行了failover本地配置的监听。

ClientWorker.checkLocalConfig判断了当前CacheData是否需要使用failover本地配置，这类配置不会从服务端获取，只能在文件系统中手动更新。

```java
private void checkLocalConfig(CacheData cacheData) {
    final String dataId = cacheData.dataId;
    final String group = cacheData.group;
    final String tenant = cacheData.tenant;
    File path = LocalConfigInfoProcessor.getFailoverFile(agent.getName(), dataId, group, tenant);

    // 当isUseLocalConfigInfo=false 且 failover配置文件存在时，使用failover配置文件，并更新内存中的配置
    if (!cacheData.isUseLocalConfigInfo() && path.exists()) {
        String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
        final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
        cacheData.setUseLocalConfigInfo(true);
        cacheData.setLocalConfigInfoVersion(path.lastModified());
        cacheData.setContent(content);
        return;
    }

    // 当isUseLocalConfigInfo=true 且 failover配置文件不存在时，不使用failover配置文件
    if (cacheData.isUseLocalConfigInfo() && !path.exists()) {
        cacheData.setUseLocalConfigInfo(false);
        return;
    }

    // 当isUseLocalConfigInfo=true 且 failover配置文件存在时，使用failover配置文件并更新内存中的配置
    if (cacheData.isUseLocalConfigInfo() && path.exists() && cacheData.getLocalConfigInfoVersion() != path
            .lastModified()) {
        String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
        final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
        cacheData.setUseLocalConfigInfo(true);
        cacheData.setLocalConfigInfoVersion(path.lastModified());
        cacheData.setContent(content);
    }
}
```



**当文件系统指定路径下的failover配置文件存在时，就会优先使用failover配置文件；当failover配置文件被删除时，又会切换为使用server端配置**。同时，如果使用failover配置文件，此处会更新CacheData中的配置。

#### 向Server端发起长轮询请求

针对所有非failover配置，通过ClientWorker.checkUpdateDataIds发起长轮询请求。

这里会统计所有非failover配置，并拼接请求业务报文：

- 有namespace的CacheData：dataId group md5 namespace
- 无namespace的CacheData：dataId group md5

此外，这里过滤出了正在初始化的CacheData，即CacheData刚构建，内部content仍然是本地snapshot版本，这部分配置将会有特殊处理。

```java
List<String> checkUpdateDataIds(List<CacheData> cacheDatas, List<String> inInitializingCacheList) throws Exception {
    StringBuilder sb = new StringBuilder();
    // 统计所有非failover的cacheData，拼接为"dataId group md5"或"dataId group md5 namespace"
    for (CacheData cacheData : cacheDatas) {
        if (!cacheData.isUseLocalConfigInfo()) {
            sb.append(cacheData.dataId).append(WORD_SEPARATOR);
            sb.append(cacheData.group).append(WORD_SEPARATOR);
            if (StringUtils.isBlank(cacheData.tenant)) {
                sb.append(cacheData.getMd5()).append(LINE_SEPARATOR);
            } else {
                sb.append(cacheData.getMd5()).append(WORD_SEPARATOR);
                sb.append(cacheData.getTenant()).append(LINE_SEPARATOR);
            }
            // 将首次监听的CacheData放入inInitializingCacheList
            if (cacheData.isInitializing()) {
                inInitializingCacheList
                        .add(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant));
            }
        }
    }
    boolean isInitializingCacheList = !inInitializingCacheList.isEmpty();
    // 实际发起请求
    return checkUpdateConfigStr(sb.toString(), isInitializingCacheList);
}
```

checkUpdateConfigStr方法负责调用服务端**/v1/cs/configs/listener**长轮询接口，并解析报文返回。关注几个点：

- 请求参数Listening-Configs是上面拼接的业务报文
- 长轮询超时时间默认30s，放在请求头Long-Pulling-Timeout里
- 如果本次长轮询包含首次监听的配置项，在请求头设置Long-Pulling-Timeout-No-Hangup=true，让服务端立即返回本次轮询结果
- 服务端/v1/cs/configs/listener接口负责处理长轮询请求
- parseUpdateDataIdResponse方法会解析服务端返回报文



```java
// ClientWorker
List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) throws Exception {
    Map<String, String> params = new HashMap<String, String>(2);
    // 拼接的业务报文 key = Listening-Configs
    params.put(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);
    Map<String, String> headers = new HashMap<String, String>(2);
    // 长轮询超时时间30s
    headers.put("Long-Pulling-Timeout", "" + timeout);
    // 告诉服务端，本次长轮询包含首次监听的配置项，不要hold住请求，立即返回
    if (isInitializingCacheList) {
        headers.put("Long-Pulling-Timeout-No-Hangup", "true");
    }
    // 如果没有需要监听的
    if (StringUtils.isBlank(probeUpdateString)) {
        return Collections.emptyList();
    }

    try {
        // timeout = 30
        // readTimeout = 30 + 15  配置中心服务端最多hold住请求30s
        long readTimeoutMs = timeout + (long) Math.round(timeout >> 1);
        // 请求/v1/cs/configs/listener
        HttpRestResult<String> result = agent
                .httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params, agent.getEncode(),
                        readTimeoutMs);

        if (result.ok()) {
            setHealthServer(true);
            // 解析返回报文
            return parseUpdateDataIdResponse(result.getData());
        } else {
            setHealthServer(false);
        }
    } catch (Exception e) {
        setHealthServer(false);
        throw e;
    }
    return Collections.emptyList();
}
```



parseUpdateDataIdResponse解析服务端返回报文，**每行报文代表一个发生配置变化的groupKey**。

```java
private List<String> parseUpdateDataIdResponse(String response) {
    if (StringUtils.isBlank(response)) {
        return Collections.emptyList();
    }
    response = URLDecoder.decode(response, "UTF-8");
    List<String> updateList = new LinkedList<String>();
    // 按行分割
    for (String dataIdAndGroup : response.split(LINE_SEPARATOR)) {
        if (!StringUtils.isBlank(dataIdAndGroup)) {
            // 每行按空格分割，拼接为dataId+group+namespace 或 dataId+group
            String[] keyArr = dataIdAndGroup.split(WORD_SEPARATOR);
            String dataId = keyArr[0];
            String group = keyArr[1];
            if (keyArr.length == 2) {
                updateList.add(GroupKey.getKey(dataId, group));
            } else if (keyArr.length == 3) {
                String tenant = keyArr[2];
                updateList.add(GroupKey.getKeyTenant(dataId, group, tenant));
            } else {
                LOGGER.error();
            }
        }
    }
    return updateList;
}
```



#### 校验md5变化并触发监听器

收到服务端返回发生变化的配置项后，客户端会通过**/v1/cs/configs接口获取对应的配置，并将配置保存到本地文件系统作为snapshot**，这部分在配置查询部分看过，都是调用ClientWorker#getServerConfig方法。最后会将配置更新到CacheData的content字段中。

上述步骤处理完成后，通过CacheData.checkListenerMd5校验配置是否发生变更，并触发监听器。



```java
// CacheData.java
// 注册在这个CacheData配置上的监听器
private final CopyOnWriteArrayList<ManagerListenerWrap> listeners;
// 配置的md5
private volatile String md5;
void checkListenerMd5() {
    for (ManagerListenerWrap wrap : listeners) {
        // 比较CacheData中的md5与Listener中上次的md5是否相同
        if (!md5.equals(wrap.lastCallMd5)) {
            // 不相同则触发监听器
            safeNotifyListener(dataId, group, content, type, md5, wrap);
        }
    }
}
```



safeNotifyListener方法是通知监听器的主逻辑，**如果Listener配置了自己的Executor，将在自己配置的线程服务里执行监听逻辑，默认使用长轮询线程执行监听逻辑**。

```java
// CacheData.java
private void safeNotifyListener(final String dataId, final String group, final String content, final String type,
        final String md5, final ManagerListenerWrap listenerWrap) {
    final Listener listener = listenerWrap.listener;

    Runnable job = new Runnable() {
        @Override
        public void run() {
            try {
              	// 如果是AbstractSharedListener，把dataId和group放到它的成员变量里
                if (listener instanceof AbstractSharedListener) {
                    AbstractSharedListener adapter = (AbstractSharedListener) listener;
                    adapter.fillContext(dataId, group);
                }

                ConfigResponse cr = new ConfigResponse();
                cr.setDataId(dataId);
                cr.setGroup(group);
                cr.setContent(content);
                // 给用户的钩子，忽略
                configFilterChainManager.doFilter(null, cr);
                String contentTmp = cr.getContent();
                // 触发监听器的receiveConfigInfo方法
                listener.receiveConfigInfo(contentTmp);

                // 如果是AbstractConfigChangeListener实例，触发receiveConfigChange方法
                if (listener instanceof AbstractConfigChangeListener) {
                    Map data = ConfigChangeHandler.getInstance()
                            .parseChangeData(listenerWrap.lastContent, content, type);
                    ConfigChangeEvent event = new ConfigChangeEvent(data);
                    ((AbstractConfigChangeListener) listener).receiveConfigChange(event);
                    listenerWrap.lastContent = content;
                }
                // 更新监听器的上次md5值
                listenerWrap.lastCallMd5 = md5;
            } catch (NacosException ex) {
                LOGGER.error();
            } catch (Throwable t) {
                LOGGER.error();
            } finally {
                Thread.currentThread().setContextClassLoader(myClassLoader);
            }
        }
    };

    try {
        // 如果监听器配置了executor，使用配置的executor执行上面的任务
        if (null != listener.getExecutor()) {
            listener.getExecutor().execute(job);
        } else {
            // 否则直接执行，也就是在长轮询线程中执行
            job.run();
        }
    } catch (Throwable t) {
        LOGGER.error();
    }
}
```

这里根据Listener的类型会触发不同的监听方法。**如果是普通的Listener，会触发receiveConfigInfo方法，传入一个String，是变更后的配置值**。

```java
public interface Listener {
    Executor getExecutor();
    void receiveConfigInfo(final String configInfo);
}
```

但是AbstractConfigChangeListener监听是有前提条件的，配置文件必须是yaml格式或properties格式，否则将不会触发Listener逻辑！见ConfigChangeHandler的parseChangeData方法，如果找不到解析器，会返回一个空的map。

```java
public Map parseChangeData(String oldContent, String newContent, String type) throws IOException {
    for (ConfigChangeParser changeParser : this.parserList) {
        // 判断是否有可以解析这种配置文件类型，目前仅支持properties和yaml
        if (changeParser.isResponsibleFor(type)) {
            return changeParser.doParse(oldContent, newContent, type);
        }
    }
    
    return Collections.emptyMap();
}
```



safeNotifyListener这部分逻辑中构造的ConfigChangeEvent将不会包含任何数据。

```java
if (listener instanceof AbstractConfigChangeListener) {
    Map data = ConfigChangeHandler.getInstance()
            .parseChangeData(listenerWrap.lastContent, content, type);
    // 如果map为空，这里构造的event里的数据就也为空了，监听器感知不到配置变更
    ConfigChangeEvent event = new ConfigChangeEvent(data);
    ((AbstractConfigChangeListener) listener).receiveConfigChange(event);
    listenerWrap.lastContent = content;
}
```



```java
public class ConfigChangeEvent {
    private final Map<String, ConfigChangeItem> data;
    public ConfigChangeEvent(Map<String, ConfigChangeItem> data) {
        this.data = data;
    }
    public ConfigChangeItem getChangeItem(String key) {
        return data.get(key);
    }
    public Collection<ConfigChangeItem> getChangeItems() {
        return data.values();
    }
}
```

# 总结

- 如何使用：直接使用ConfigService即可实现配置增删改查和监听。
- Nacos配置模型
  - **Namspace（Tenant）：命名空间（租户）**，默认命名空间是public。一个命名空间可以包含多个Group，在Nacos源码里有些变量是tenant租户，和命名空间是一个东西。
  - **Group：组**，默认分组是DEFAULT_GROUP。一个组可以包含多个dataId。
  - **DataId：译为数据id，在nacos中DataId代表一整个配置文件，是配置的最小单位**。
- 一个nacos-server+一个namespace对应一个ConfigService。
- Nacos客户端查询配置
  - 优先failover本地配置
  - 其次调用服务端查询/v1/cs/configs
  - 兜底使用snapshot本地配置，每次成功调用服务端/v1/cs/configs获取配置，都会同步更新snapshot配置
- Nacos客户端监听配置，通过LongPollingRunnable处理，对于非failover配置项，处理流程如下图。

![客户端长轮询步骤.png](../images/72612155f8e14c2d9907b0e4e337bea7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215844476.awebp)

