# 认识 Zookeeper




## 初识 Zookeeper

### ZooKeeper 概念

- Zookeeper 是 Apache Hadoop 项目下的一个子项目，是一个树形目录服务。
- Zookeeper 翻译过来就是 动物园管理员，他是用来管 Hadoop（大象）、Hive(蜜蜂)、Pig(小猪)的管理员。简称zk
- Zookeeper 是一个分布式的、开源的分布式应用程序的协调服务。
- Zookeeper 提供的主要功能包括：
  - 配置管理
  - 分布式锁
  - 集群管理

### Zookeeper 数据模型

ZooKeeper 是一个树形目录服务,其数据模型和Unix的文件系统目录树很类似，拥有一个层次化结构。
这里面的每一个节点都被称为： ZNode，每个节点上都会保存自己的数据和节点信息。
节点可以拥有子节点，同时也允许少量（1MB）数据存储在该节点之下。
节点可以分为四大类：

- 持久化节点（PERSISTENT）
- 临时节点（EPHEMERAL）：-e
- 持久化顺序节点（PERSISTENT_SEQUENTIAL）：-s
- 临时顺序节点（EPHEMERAL_SEQUENTIAL）：-es

### Docker安装Zookeeper

1. 查看本地镜像和检索拉取Zookeeper 镜像

   ```lua
   # 查看本地镜像
   docker images
   # 检索ZooKeeper 镜像
   docker search zookeeper
   # 拉取ZooKeeper镜像最新版本
   docker pull zookeeper:latest
   ```

2. 创建ZooKeeper 挂载目录（数据挂载目录、配置挂载目录和日志挂载目录）

   mkdir -p /mydata/zookeeper/data # 数据挂载目录
   mkdir -p /mydata/zookeeper/conf # 配置挂载目录
   mkdir -p /mydata/zookeeper/logs # 日志挂载目录

3. 启动ZooKeeper容器

   ```dockerfile
   docker run -d --name zookeeper --privileged=true -p 2181:2181  -v D:/Software/Docker/Zookeeper/data:/data -v D:/Software/Docker/Zookeeper/conf:/conf -v D:/Software/Docker/Zookeeper/logs:/datalog zookeeper:latest
   ```

   参数说明

   ```do
   -e TZ="Asia/Shanghai" # 指定上海时区 
   -d # 表示在一直在后台运行容器
   -p 2181:2181 # 对端口进行映射，将本地2181端口映射到容器内部的2181端口
   --name # 设置创建的容器名称
   -v # 将本地目录(文件)挂载到容器指定目录；
   --restart always #始终重新启动zookeeper，看需求设置不设置自启动
   ```

4. 添加ZooKeeper配置文件，在挂载配置文件目录(/mydata/zookeeper/conf)下，新增zoo.cfg 配置文件，配置内容如下：

   ```dockerfile
   dataDir=/data  # 保存zookeeper中的数据
   clientPort=2181 # 客户端连接端口，通常不做修改
   dataLogDir=/datalog
   tickTime=2000  # 通信心跳时间
   initLimit=5    # LF(leader - follower)初始通信时限
   syncLimit=2    # LF 同步通信时限
   autopurge.snapRetainCount=3
   autopurge.purgeInterval=0
   maxClientCnxns=60
   standaloneEnabled=true
   admin.enableServer=true
   server.1=localhost:2888:3888;2181
   ```

5. 进入容器内部，验证容器状态

   ```dockerfile
   # 进入zookeeper 容器内部
   docker exec -it zookeeper /bin/bash
   # 检查容器状态
   docker exec -it zookeeper /bin/bash ./bin/zkServer.sh status
   # 进入控制台
   docker exec -it zookeeper zkCli.sh
   ```

## ZooKeeper JavaAPI 操作

### Curator 介绍

Curator 是 Apache ZooKeeper 的Java客户端库。
常见的ZooKeeper Java API ：

- 原生Java API
- ZkClient
- Curator

Curator 项目的目标是简化 ZooKeeper 客户端的使用。
Curator 最初是 Netfix 研发的,后来捐献了 Apache 基金会,目前是 Apache 的顶级项目。

### Curator API 常用操作

1. 建立连接

   引入依赖

   ```java
   	<!--curator-->
           <dependency>
               <groupId>org.apache.curator</groupId>
               <artifactId>curator-framework</artifactId>
               <version>4.0.0</version>
           </dependency>
   
           <dependency>
               <groupId>org.apache.curator</groupId>
               <artifactId>curator-recipes</artifactId>
               <version>4.0.0</version>
           </dependency>
   ```

   创建测试类

   ```java
   import org.apache.curator.RetryPolicy;
   import org.apache.curator.framework.CuratorFramework;
   import org.apache.curator.framework.CuratorFrameworkFactory;
   import org.apache.curator.retry.ExponentialBackoffRetry;
   
   public class CuratorTest {
   
       public CuratorFramework getConnect() {
   
           /**
            * @param connectString         连接字符串，zk server地址和端口 "127.0.0.1:2181"
            * @param sessionTimeoutMs      会话超时时间，单位是毫秒ms
            * @param connectionTimeoutMs   连接超时时间，单位是毫秒ms
            * @param retryPolicy           重试策略
            */
           /*
           //重试策略
           RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000, 10);
           //第一种方式
           CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 60 * 1000, 15 * 1000, retryPolicy);
           //开启连接
           client.start();
           */
           //第二种方式,链式创建
           RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000, 10);
           CuratorFramework curatorFramework = CuratorFrameworkFactory.builder()
                   .connectString("127.0.0.1:2181")
                   .sessionTimeoutMs(60 * 1000)
                   .connectionTimeoutMs(15 * 1000)
                   .retryPolicy(retryPolicy).namespace("smsService").build();
   
           curatorFramework.start();
           return curatorFramework;
       }
   }
   ```

2. 添加节点

   ```java
   public void createNode() throws Exception {
           CuratorFramework client = getConnect();
       
           //基本创建
           String path = client.create().forPath("/sztus");
           System.out.println(path);
   
           //带数据的创建
           client.create().forPath("/sztus","jerry_test".getBytes(StandardCharsets.UTF_8));
   
           //设置节点类型,临时类型，默认类型持久化
           client.create().withMode(CreateMode.EPHEMERAL).forPath("/sztus");
   
           //创建多级节点  /app4/ios
           //creatingParentsIfNeeded,如果父节点不存在则创建父节点
           client.create().creatingParentsIfNeeded().forPath("/sztus/azeroth");
   
           client.close();
       }
   ```

3. 删除节点

   ```java
   public void deleteNode() throws Exception {
           CuratorFramework client = getConnect();
   
           //删除单个节点
           client.delete().forPath("/sztus");
           //删除带有子节点的节点
           client.delete().deletingChildrenIfNeeded().forPath("/sztus");
           //必须删除成功
           client.delete().guaranteed().forPath("/sztus");
           //回调
           client.delete().guaranteed().inBackground(new BackgroundCallback() {
               @Override
               public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {
                   System.out.println("被删除了" + event);
               }
           }).forPath("/sztus");
   
           client.close();
       }
   ```

4. 修改节点

   ```java
   public void updateNode() throws Exception {
           CuratorFramework client = getConnect();
   
           //简单修改数据
           client.setData().forPath("/sztus", "sztus_test".getBytes(StandardCharsets.UTF_8));
   
           //判断版本信息一致再修改
           Stat stat = new Stat();
           client.getData().storingStatIn(stat).forPath("/sztus");
           int version = stat.getVersion();
           client.setData().withVersion(version).forPath("/sztus", "sztus_test".getBytes(StandardCharsets.UTF_8));
   
           client.close();
       }
   ```

5. 查询节点

   ```java
   public void getNode() throws Exception {
           CuratorFramework client = getConnect();
           //查询数据  get
           byte[] bytes = client.getData().forPath("/sztus");
           System.out.println(new String(bytes));
   
           //查询子节点  ls
           List<String> path = client.getChildren().forPath("/");
           System.out.println(path);
   
           //查询节点状态信息  ls -s
           Stat stat = new Stat();
           client.getData().storingStatIn(stat).forPath("/sztus");
           System.out.println("====" +stat);
   
           client.close();
       }
   ```

6. Watch事件监听

   ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。
   ZooKeeper 中引入了 Watcher 机制来实现了发布/订阅功能能，能够让多个订阅者同时监听某一个对象，当一个对象自身状态变化时，会通知所有订阅者。
   ZooKeeper 原生支持通过注册 Watcher来进行事件监听，但是其使用并不是特别方便
   需要开发人员自己反复注册 Watcher，比较繁琐。
   Curator 引入了 Cache 来实现对 ZooKeeper 服务端事件的监听。

   - NodeCache：只是监听某一个特点的节点

     ```java
     public void watchNode() {
         CuratorFramework client = getConnect();
     
         CuratorCache cache = CuratorCache.build(client, "/sztus/dolphin",
                 CuratorCache.Options.SINGLE_NODE_CACHE);
         CuratorCacheListener listener = CuratorCacheListener.builder()
                 .forNodeCache(() -> System.out.println("node change"))
                 .build();
         
         client.close();
     }
     ```

   - PathChildrenCache : 监控一个 ZNode 的子节点.，感知不到自己的变化。

     ```java
     public void watchChildrenNode() {
         CuratorFramework curator = getConnect();
     
         CuratorCache cache = CuratorCache.build(curator, "/sztus",
                 CuratorCache.Options.SINGLE_NODE_CACHE);
     
         CuratorCacheListener listener = CuratorCacheListener.builder()
                 .forPathChildrenCache("/sztus", curator, (client, event) -> {
                     System.out.println(event);
                 })
                 .build();
     
         curator.close();
     }
     ```

   - TreeCache : 可以监控整个树上的所有节点，类似于 PathChildrenCache 和 NodeCache 的组合。

     ```java
     public void watchChildrenNode() {
         CuratorFramework curator = getConnect();
     
         CuratorCache cache = CuratorCache.build(curator, "/sztus",
                 CuratorCache.Options.SINGLE_NODE_CACHE);
         CuratorCacheListener listener = CuratorCacheListener.builder()
                 .forTreeCache(curator, (client, event) -> {
                     System.out.println(event);
                 })
                 .build();
     
         curator.close();
     }
     ```

7. Zookeeper实现分布式锁

   核心思想：当客户端要获取锁，则创建节点，使用完锁，则删除该节点。

   - 客户端获取锁时，在 lock 节点下创建临时顺序节点。
   - 然后获取 lock 下面的所有子节点，客户端获取到所有的子节点之后，如果发现自己创建的子节点序号最小，那么就认为该客户端获取到了锁。使用完锁后，将该节点删除。
   - 如果发现自己创建的节点并非lock所有子节点中最小的，说明自己还没有获取到锁，此时客户端需要找到比自己小的那个节点，同时对其注册事件监听器，监听删除事件。
   - 如果发现比自己小的那个节点被删除，则客户端的 Watcher 会收到相应通知，此时再次判断自己创建的节点是否是 lock 子节点中序号最小的，如果是则获取到了锁，如果不是则重复以上步骤继续获取到比自己小的一个节点并注册监听。

   在 Curator 中有五种锁方案：

   - InterProcessSemaphoreMutex：分布式排它锁（非可重入锁）
   - InterProcessMutex：分布式可重入排它锁
   - InterProcessReadWriteLock：分布式读写锁
   - InterProcessMultiLock：将多个锁作为单个实体管理的容器
   - InterProcessSemaphoreV2：共享信号量

   这里把 Zookeeper 与 Redis 实现分布式锁对比一下：

   - 优点：ZooKeeper分布式锁（如 InterProcessMutex），除了独占锁、可重入锁，还能实现读写锁，并且可靠性比 Redis 更好。
   - 缺点：ZooKeeper实现的分布式锁，性能并不太高。因为每次在创建锁和释放锁的过程中，都要动态创建、销毁瞬时节点来实现锁功能。而 ZK 中创建和删除节点只能通过 Leader 服务器来执行，然后 Leader 服务器还需要将数据同不到所有的 Follower 机器上，同步之后才返回，这样频繁的网络通信，性能的短板是非常突出的；而 Redis 则是异步复制。

   Redis 是 AP 架构，而 ZooKeeper 是 CP 架构。在高性能，高并发的场景下，不建议使用ZooKeeper的分布式锁，可以使用 Redis 分布式锁。而由于ZooKeeper的可靠性，所以在并发量不是太高的场景，推荐使用ZooKeeper的分布式锁。

## Zookeeper服务注册与发现

更改 pom 文件，增加 Zookeeper 的依赖

```java
<dependencies>
        <!--   用于读取 bootstrap.yml   -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bootstrap</artifactId>
        </dependency>

        <!-- Sprint Boot Web Component -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Sprint Boot Unit Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    
        <!-- SpringBoot整合zookeeper客户端 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>
```

修改`bootstrap.yml`文件

```yaml
# 9555表示注册到 zookeeper 服务器的支付服务提供者端口号
server:
  port: 9555
# 服务别名---- 注册 zookeeper 到注册中心名称
spring:
  application:
    name: cloud-zookeeper-server
  cloud:
    zookeeper:
      # 如果 zookeeper 是集群，指定多个地址即可：x.x.x.x:2181, y.y.y.y:2181
      connect-string: localhost:2181
```

## Zookeeper的分布式配置中心与监控
