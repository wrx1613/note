# nacos集群部署

nacos版本：1.3.0

mysql数据库：5.7.35

### 1.下载

官网：https://github.com/alibaba/nacos/releases/tag/1.3.0

ftp：服务源文件：ftp://192.168.6.163/环境搭建/nacos-server-1.3.0.tar.gz

​         已配置的集群：ftp://192.168.6.163/环境搭建/nacos.zip

### 2.初始化数据库

mysql安装教程 略

创建数据库并初始化，数据库脚本位置：conf/nacos-mysql.sql

修改数据库配置，配置文件位置：conf/application.properties

```bash
### If user MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
# 多个数据库实例
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
# db.url.1=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=123456
```

### 3.nacos端口配置

配置文件位置：conf/application.properties

```
### Default web server port:
server.port=8847
```

附：该配置可用于修改主页路径

```
### Default web context path:
server.servlet.contextPath=/nacos
```

### 4.集群配置

创建文件 conf/application.properties 并按如下格式配置自己的集群节点

```
192.168.6.163:8847
192.168.6.163:8848
192.168.6.163:8849
```

### 5.启动

将所有集群依次启动

```
sh bin/startup.sh
```

### 6.访问

访问地址： http://{任意的nacos地址}/nacos

# FAQ

#### go客户端如何保证总是连接可用的nacos节点

go-nacos客户端访问服务端采用http连接

当客户端配置仅一个节点时，如果访问失败，会重新访问，3次之后还是失败返回err

```
for i := 0; i < constant.REQUEST_DOMAIN_RETRY_TIME; i++ {
    ...
}

# common/constant/const.go
REQUEST_DOMAIN_RETRY_TIME = 3
```

当客户端配置多个节点时，会随机访问其中一个，如果访问失败，则会继续访问下一个，如果成功则返回，否则直至所有节点都访问完毕返回err

```
index := rand.Intn(len(srvs))
for i := 1; i <= len(srvs); i++ {
    ...
    if err == nil {
		return
	}
    index = (index + i) % len(srvs)
}
```

#### 如果手动删除临时节点后，客户端能否检测到并重新创建

能。

以go-nacos客户端为例，在注册服务时如果是临时实例，会开启一个定时任务向服务端发送心跳，服务端接收到心跳时该实例不存在则会创建该实例。

go-nacos注册服务实例代码

```
// 注册服务实例
func (sc *NamingClient) RegisterInstance(param vo.RegisterInstanceParam) (bool, error) {
	...
	if instance.Ephemeral {
		//添加定时任务，每5s执行一次，发送心跳
		sc.beatReactor.AddBeatInfo(util.GetGroupName(param.ServiceName, param.GroupName), beatInfo)
	}
	return true, nil
}
```

nacos服务端接收心跳代码

```
@CanDistro
@PutMapping("/beat")
@Secured(parser = NamingResourceParser.class, action = ActionTypes.WRITE)
public ObjectNode beat(HttpServletRequest request) throws Exception {
	...
	if (instance == null) {
		...
		//注册实例
		serviceManager.registerInstance(namespaceId, serviceName, instance);
	}
	...
}
```

#### nacos临时节点在集群间的同步

对所访问 nacos 节点对服务实例增删改后，该节点会重新把数据同步给其他节点

```
# com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl.java

@Override
public void put(String key, Record value) throws NacosException {
	onPut(key, value);
	taskDispatcher.addTask(key); //把同步数据的任务放入队列
}
```

从线程池中随机取出一个线程并放入任务

```
# com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher.java
public void addTask(String key) {
	taskSchedulerList.get(UtilsAndCommons.shakeUp(key, cpuCoreCount)).addTask(key);
}

# taskSchedulerList 启动时初始化的一个线程池
# UtilsAndCommons.shakeUp(key, cpuCoreCount)  根据key返回0到cpuCoreCount间的一个数字
```

线程池中的线程执行任务如下

```
# com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher.java#TaskScheduler

@Override
public void run() {
	if ( 当任务数超过1000或时间间隔超过2s ) {
        // 遍历所有nacos节点
        for (Member member : dataSyncer.getServers()) {
            // 排除当前节点
            if (NetUtils.localServer().equals(member.getAddress())) {
                continue;
            }
            // 封装任务
            SyncTask syncTask = new SyncTask();
            syncTask.setKeys(keys);
            syncTask.setTargetServer(member.getAddress());
            // 提交任务
            dataSyncer.submit(syncTask, 0);
        }
	}
}
```

执行同步任务

```
# com.alibaba.nacos.naming.consistency.ephemeral.distro.DataSyncer.java

public void submit(SyncTask task, long delay) {
	...
    GlobalExecutor.submitDataSync(() -> {
        // 1. check the server
        ...
        // 2. get the datums by keys and check the datum is empty or not
        ...
        long timestamp = System.currentTimeMillis();
        // 同步数据
        boolean success = NamingProxy.syncData(data, task.getTargetServer());
        if (!success) {
        	// 失败则重新提交同步任务
            SyncTask syncTask = new SyncTask();
            syncTask.setKeys(task.getKeys());
            syncTask.setRetryCount(task.getRetryCount() + 1);
            syncTask.setLastExecuteTime(timestamp);
            syncTask.setTargetServer(task.getTargetServer());
            retrySync(syncTask);
        } else {
            // clear all flags of this task:
            for (String key : task.getKeys()) {
            	taskMap.remove(buildKey(key, task.getTargetServer()));
            }
        }
    }, delay);
}
```

同步数据，http请求

```
# com.alibaba.nacos.naming.misc.NamingProxy.java

public static boolean syncData(byte[] data, String curServer) {
    Map<String, String> headers = new HashMap<>(128);
    headers.put(HttpHeaderConsts.CLIENT_VERSION_HEADER, VersionUtils.VERSION);
    headers.put(HttpHeaderConsts.USER_AGENT_HEADER, UtilsAndCommons.SERVER_VERSION);
    headers.put("Accept-Encoding", "gzip,deflate,sdch");
    headers.put("Connection", "Keep-Alive");
    headers.put("Content-Encoding", "gzip");
    try {
        HttpClient.HttpResult result = HttpClient.httpPutLarge("http://" + curServer + ApplicationUtils.getContextPath()
        	+ UtilsAndCommons.NACOS_NAMING_CONTEXT + DATA_ON_SYNC_URL, headers, data);
        if (HttpURLConnection.HTTP_OK == result.code) {
            return true;
        }
        if (HttpURLConnection.HTTP_NOT_MODIFIED == result.code) {
        	return true;
        }
        throw new IOException("failed to req API:" + "http://" + curServer
            + ApplicationUtils.getContextPath()
            + UtilsAndCommons.NACOS_NAMING_CONTEXT + DATA_ON_SYNC_URL + ". code:"
            + result.code + " msg: " + result.content);
    } catch (Exception e) {
   		Loggers.SRV_LOG.warn("NamingProxy", e);
    }
    return false;
}
```

#### 什么是保护阈值

保护阈值是 当前服务健康实例数/当前服务总实例数 的一个浮点数，大小在0-1之间

⼀般情况下，服务端返回给客户端的实例都是健康的，当健康实例很少的时候遇上⼤流量可能会造成仅有的实例挂掉。

保护阈值的意义在于当 健康实例数/总实例数 < 保护阈值 的时候，服务端把所有的实例信息（健康的+不健康的）全部提供给客户端，从而客户端可能访问到不健康的实例，请求失败。

牺牲了⼀些请求，保证了整个系统的⼀个可⽤性

#### nacos内存配置

内存配置位于启动脚本 startup.cmd / startup.sh

```
...
if [[ "${MODE}" == "standalone" ]]; then
    JAVA_OPT="${JAVA_OPT} -Xms512m -Xmx512m -Xmn256m"   #这是单机模式配置
    JAVA_OPT="${JAVA_OPT} -Dnacos.standalone=true"
else
    if [[ "${EMBEDDED_STORAGE}" == "embedded" ]]; then
        JAVA_OPT="${JAVA_OPT} -DembeddedStorage=true"
    fi
    JAVA_OPT="${JAVA_OPT} -server -Xms2g -Xmx2g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"  #这是集群模式配置
    JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${BASE_DIR}/logs/java_heapdump.hprof"
    JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages"
...
```

说明

-Xms: 设定程序启动时占用内存大小

-Xmx: 设定程序运行期间最大可占用的内存大小

-Xmn：新生代大小

