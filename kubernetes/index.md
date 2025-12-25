# Kubernetes 的注册中心和配置中心(Etcd))


## 什么是ETCD?

[`Etcd`](https://etcd.io/)是一个基于`go`语音实现，用于共享配置和服务发现的分布式储存系统，在分布式系统中提供强一致性、高可用性的功能。它提供了一种可靠的方式来存储需要被分布式系统或机器集群访问的数据。`Etcd`操作简单，使用`Http`请求的方式，对数据读取和写入，它将数据存储在分层组织的目录中；并且监听特定的`key`或者目录，根据需求做出相应的反应。

基于`Etcd`提供的`api`，可以实现以下几个功能：

- 服务注册与发现
- 配置管理
- 消息发布和订阅
- 负载均衡
- 分布式锁、分布式队列
- 集群监控
- leader选举（基于raft算法）

`Etcd`主要分为4个部分：

- `Http Server`：用于处理用户发送的`API`请求以及其它`Etcd`节点的同步和心跳信息请求。
- `Store`：用于处理`Etcd`支持的各类功能事务，包括数据索引、节点状态更新、监控与反馈、事件处理与执行等等，是`etcd`对用户提供的大多数`API`功能的具体实现。
- `Raft`：`Raft`是强一致性算法的具体实现，是`Etcd`的核心。
- `WAL`：`Write Ahead Log`（预写式日志），是`Etcd`的数据储存方式。除了在内存中存有所有数据的状态以及节点的索引以外，`Etcd`就通过`WAL`进行持久化存储。`WAL`中，所有的数据提交前都会事先记录日志。`Snapshot`是为了防止数据过多而进行的状态快照；`Entry`表示存储的具体日志内容。

<img src="C:\Users\Jerry\AppData\Roaming\Typora\typora-user-images\image-20230328175336446.png" alt="image-20230328175336446" style="margin-left:20%" />

## 环境准备

- WSL(Windows Subsystem for Linux)

  为使用`Kind`准备环境，让环境虚拟化

- Kind(Kubernetes In Docker)

  下载`Kind` ，[下载地址](https://github.com/kubernetes-sigs/kind/releases/)。`Kind`是绿色软件，下载后改名 `kind.exe`放到 `C:\Windows\`目录下即可。

- Kubectl

  `kubectl`是管理`Kubernetes`集群的命令行工具，下载`kubectl`，[下载地址](https://kubernetes.io/docs/tasks/tools)。下载后放到 `C:\Windows\`目录下即可。

- Docker

  安装`Docker`，[官网下载安装包](https://www.docker.com/products/docker-desktop)，一路下一步安装即可。安装之后，需要打开一次来确认安装是否成功。

## Kubernetes的注册中心

### 注册中心对比

目前`java`生态中，主流的注册中心有：`Eureka`、`Nacos`、`Zookeeper`、`Consul`、`Etcd`。其中，`Etcd`和`Consul`是`Go`语言开发的，`Eureka`、`Nacos`、`Zookeeper`都是`Java`开发的。目前这5个注册中心都有相对应的`boot start`，提供了集成能力。

<img src="C:\Users\Jerry\AppData\Roaming\Typora\typora-user-images\image-20230328170109604.png" alt="image-20230328170109604" style="float:left" />

## Spring微服务集成Kubernetes的注册中心

`maven`依赖配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
    <version>${kubernetes}</version>
</dependency>
```

服务的`yml`配置

```yaml
szt:
  service:
    name: azeroth-provider
    port: 8811
server:
  port: ${szt.service.port}
spring:
  application:
    name: ${szt.service.name}
```

在启动类上加`EnableDiscoveryClient`注解

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AzerothProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(AzerothProviderApplication.class, args);
    }

}
```

添加`Controller`类，使用`DiscoveryClient`获取服务及其参数

```java
@RestController
public class ProviderController {

    @Autowired
    private DiscoveryClient discoveryClient;

    /**
     * 获取所有服务id
     */
    @GetMapping("/service")
    public List<String> getServiceList() {
        return discoveryClient.getServices();
    }

    /**
     * 获取服务信息
     */
    @GetMapping("/instance")
    public Object getInstance(@RequestParam("name") String name) {
        return discoveryClient.getInstances(name);
    }

    /**
     * 获取所有服务及其信息
     */
    @GetMapping("/service-information")
    public List<List<ServiceInstance>> getServiceInformation() {
        List<String> list = discoveryClient.getServices();
        List<List<ServiceInstance>> result = new ArrayList<List<ServiceInstance>>();
        for (String name : list) {
            List<ServiceInstance> instances = discoveryClient.getInstances(name);
            result.add(instances);
        }
        return result;
    }

    @GetMapping("/hello")
    public String test() {
        return "Provider Server";
    }
}
```

`DiscoveryClient`是`Spring Cloud Commons`提供的一个接口，负责服务的发现和注册，`Kubernetes`有一个实现类`KubernetesInformerDiscoveryClient`，因此可以`Kubernetes`可以与`Spring Cloud`完美衔接。

将镜像加载到`kubernetes`中

```shell
kind load docker-image provider:v4 --name=aztus
```

编写`provider.yaml`，创建`deployment`控制器和对外桥接的`service`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: provider
  namespace: default
  labels:
    app: provider
spec:
  replicas: 1
  selector:
    matchLabels:
      app: provider
  template:
    metadata:
      labels:
        app: provider
    spec:
      containers:
        - name: provider
          image: docker.io/library/provider:v4
          env:
            - name: spring.profiles.active
              value: develop
          ports:
            - name: rest
              containerPort: 8811
              hostPort: 8811
---
apiVersion: v1
kind: Service
metadata:
  name: provider
  namespace: default
  labels:
    app: provider
spec:
  type: NodePort
  selector:
    app: provider
  ports:
    - name: rest
      protocol: TCP
      port: 8811
      targetPort: 8811
      nodePort: 30001
```

部署`deployment`和`service`

```shell
kubectl apply -f provider.yaml
```

调用接口，查看`kubernetes`注册中心中的服务以及服务的详细信息

```shell
http://localhost:30001/service
[
    "provider",
    "kubernetes"
]

http://localhost:30001/instance?name=provider
[
    {
        "instanceId": "4a6e734a-0628-4000-8522-0d39d021aa08",
        "serviceId": "provider",
        "host": "10.244.0.6",
        "port": 8811,
        "uri": "http://10.244.0.6:8811",
        "secure": false,
        "metadata": {
            "app": "provider",
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"provider\"},\"name\":\"provider\",\"namespace\":\"default\"},\"spec\":{\"ports\":[{\"name\":\"rest\",\"nodePort\":30001,\"port\":8811,\"protocol\":\"TCP\",\"targetPort\":8811}],\"selector\":{\"app\":\"provider\"},\"type\":\"NodePort\"}}\n",
            "rest": "8811"
        },
        "namespace": "default",
        "cluster": null,
        "scheme": "http"
    }
]
```

到这里，我们就已经将我们的服务注册到`kubernetes`的注册中心上了。

### Feign的调用

接下来，我们简单说一下`Feign`的集成，看一看服务与服务之间的调用。

编写另一个服务`consumer`，服务的配置与`provider`的配置一样。`controller`层不同

```java
@RestController
@Slf4j
public class ConsumerController {

    @Autowired
    private ProviderApi providerApi;

    @GetMapping("/hello")
    public String hello() {
        log.info("[Consumer Log] hello method");
        return providerApi.hello();
    }

}
```

编写`providerApi`

```java
@FeignClient(value = "provider", path = "", contextId = "provider-api")
public interface ProviderApi {

    @GetMapping("/hello")
    String hello();
}
```

现在我们将服务部署到`kubernetes`集群中，就可以用`consumer`调用`provider`提供的服务了

```shell
kind load docker-image consumer:v4 --name=sztus
kubectl create deployment consumer --image=docker.io/library/consumer:v4 --port=8812
kubectl create service nodeport consumer --tcp=8812:8812 --node-port=30002

http://localhost:30002/hello
Provider Server
```

可以看到，我们提供`consumer`服务，成功调用到了`provider`提供的服务

## Spring微服务集成Kubernetes的配置中心（configmap）

`pom`文件的依赖与注册中心一直

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
    <version>${kubernetes}</version>
</dependency>
```

`yml`配置

```yaml
szt:
  service:
    name: azeroth-config
    port: 8813
server:
  port: ${szt.service.port}
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        mode: polling
        period: 500
      config:
        sources:
          - name: ${szt.configmap}    # szt.configmap和spring.profiles.active，设置在deployment的部署文件中
            namespace: default		  # 指定kubernetes的namespace，建议也写在deployment的部署文件中
  application:
    name: ${szt.service.name}
```

编写`controller`层的代码

```java
@RestController
public class ConfigController {

    @Autowired
    private CustomerService customerService;

    @Value("${greeting.message:null}")
    private String greeting;

    @Value("${database.url:null}")
    private String database;

    /**
     * 如果开启kubernetes的健康检查，会定时调用此接口
     */
    @GetMapping("/health")
    public String health() {
        return "success";
    }

    @GetMapping("/config/hello")
    public String hello() {
        return greeting;
    }

    @GetMapping("/config/database")
    public String database() {
        return database;
    }
}
```

通过`kind`创建kubernetes集群，并将镜像加载到集群中

```shell
kind load docker-image config:v4 --name=sztus  # 将docker镜像config，加载到sztus的kubernetes集群中
```

接下来，我们需要将我们的配置加载到`configmap`中，先准备两份`yml`文件

`develop`的配置，在`/configmap/develop`下创建`azeroth-config.yml`文件

```yaml
greeting:
  message: Hello config map for development
database:
  url: This is the link to the development database
```

`release`的配置，在`/configmap/release`下创建`azeroth-config.yml`文件

```yaml
greeting:
  message: Hello config map for production
farewell:
  message: Here is the link to the production database
```

分别将两份文件加载到两个`configmap`中

```shell
# 使用当前目录下的configmap中的develop文件夹中的所有配置，加载到configmap-develop中
kubectl create configmap configmap-develop --from-file=configmap/develop
# 使用当前目录下的configmap中的develop文件夹中的所有配置，加载到configmap-release中
kubectl create configmap configmap-release --from-file=configmap/release
```

查看加载的配置

```shell
$ kubectl get configmap  # 查看所有的configmap
NAME                   DATA   AGE
configmap-develop      1      2d3h
configmap-release      1      2d16h
kube-root-ca.crt       1      2d16h

$ kubectl describe configmap configmap-develop  # 查看某个具体的configmap信息
Name:         configmap-develop
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
azeroth-config.yml:
----
greeting:
  message: Hello config map for development
database:
  url: This is the link to the development database

BinaryData
====

Events:  <none>
```

到这一步，我们就已经把所有的配置加载到`kubernetes`的配置中心里了。接下来我们需要启动服务，去获取配置中心中的配置，先创建一个控制器。

创建控制器`deployment`的配置：`config.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config
  labels:
    app: config
spec:
  replicas: 3
  selector:
    matchLabels:
      app: config
  template:
    metadata:
      labels:
        app: config
    spec:
      containers:
        - name: config
          image: docker.io/library/config:v4  # 这里是用kind加载到集群中的镜像名
          env:
            - name: spring.profiles.active    # 设置服务的环境
              value: develop
            - name: szt.configmap			  # yml配置中的configmap参数，指定从那个configmap中读取配置文件
              value: configmap-develop
          ports:
            - containerPort: 8813
```

使用`kubectl`部署控制器`deployment`和`service`

```shell
 kubectl apply -f config.yaml  # 通过config.yaml创建或者更新deployment
 kubectl create service nodeport config --tcp=8813:8813 --node-port=30003  # 创建一个service
```

通过`service`访问接口，查看运行结果

```shell
http://localhost:30003/config/hello
Hello config map for development
```

# 附加：

## 基于系统和注册中心的特性（CAP）

本节内容，主要是探索目前主要的分布式、一致性键值数据存储软件性能的对比，通过对`Etcd`、`Zookeeper`和`Consul`多个层面的对比，我们可以全面地了解

## Kubernetes介绍

### 组件介绍

#### Namespace

`K8S`集群中默认的`Namespace`为`default`，通过`Namespace`可以实现`Pod`之间的相互隔离（如测试环境、生成环境的隔离）
通过`K8S`的资源配置机制限定不同的`Namespace`对`CPU`、内存的使用，再通过`K8S`授权机制将不同的`Namespace`交给不同的租户进行管理，实现多租户资源的隔离

`Kubernetes`默认自带的命名空间

- `default `未指定命名空间的对象都会被分配到该空间
- `kube-node-lease` 集群节点之间的心跳维护
- `kube-public `此命名空间下的资源可以被所有用户访问
- `kube-system` 由`K8S`系统创建的资源都会被分配到该空间

```shell
kubectl create ns [命名空间名称] # 创建命名空间
kubectl get ns # 查询命名空间
kubectl delete ns [命名空间名称] --force --grace-period=0 # 删除命名空间，该空间下的所有资源也将被删除
```

#### Label

`Label`通过`K-V`的形式在资源上添加标识，用它来对资源（`Node`、`Pod`、`Service`）进行区分和选择，实现资源的多纬度分组，以便灵活、方便地进行资源分配、调度、配置和部署等管理

常用的标签：`frontend`、`backend`、`dev`、`test`、`release`、`stable`

```shell
kubectl label pod [Pod名称] -n [命名空间] version=1.0 # 给Pod添加标签（version=1.0）
kubectl label pod [Pod名称] -n [命名空间] version=2.0 --overwrite # 给Pod更新标签（version=2.0）
kubectl label pod [Pod名称] -n [命名空间] version- # 给Pod删除标签（version）

kubectl get pod nginx -n [命名空间] --show-labels # 查看Pod（nginx）的标签
kubectl get pod -l "version=1.0" -n dev --show-labels # 筛选Pod中标签为version=1.0
kubectl get pod -l "version!=3.0" -n default --show-labels # 筛选Pod中标签为version!=3.0
kubectl get pod -l "version in (1.0,3.0)" -n default --show-labels # 筛选Pod中标签为version是1.0或者3.0
kubectl get pod -l "version=1.0， app=nginx" -n default --show-labels # 筛选Pod中标签为version是1.0并且app是nginx的
```

#### Pod

`Pod`是`k8s`集群进行管理的最小单元，程序运行在容器中，容器运行在于`Pod`中，一个`Pod`中可以运行一个或多个容器

Pod更新策略

- 重建更新（创建新的Pod前，会删除所有已经存在的Pod）
- 滚动更新（历史版本逐步替换新版本，最终新版本全部覆盖历史版本）

```shell
kubectl run [Pod名称] --image=[镜像名称] --port=[端口] --namespace=[命名空间] # 创建并运行一个pod（不推荐直接运行Pod）
kubectl delete pod [Pod名称] -n [命名空间] --force --grace-period=0 # 删除pod
kubectl get pod -n [命名空间] -o wide # 查询所有Pod的基本信息
kubectl describe pod [Pod名称] -n [命名空间] # 查看Pod的详细信息
kubectl logs -f [Pod名称] -n [命名空间] # 查看pod的日志
```

#### Deployment（Pod控制器）

`Deployment`用于管理无状态的应用，支持滚动升级、回退

通过`Pod`直接运行容器，会有以下缺点：

- `Pod`重建后`IP`会变化，外部无法得知最新的`IP`
- 业务应用无法启动多个副本
- 运行业务`Pod`的某个节点挂了，无法实现故障转移、恢复

`Pod`是`K8S`的最小控制单元，但`K8S`并不直接控制`Pod`，而通过`Deployment`来操作`Pod`
`Deployment`用于对`Pod`进行管理，确保`Pod`资源符合预期的状态（当`Pod`资源出现故障时，会尝试对`Pod`进行重启或重建）
通过命令直接删除`Pod`，`Pod`会在控制器的作用下重新创建并启动
通过命令直接删除`Deployment`，该控制器下的所有`Pod`都会被删除
通过命令直接删除`NameSpace`，该命名空间下的所有`Deployment`、`Pod`都会被删除

`ReplicaSet`：保证指定数量的`Pod`正常运行，一旦`Pod`发生故障就会重启或重建，同时还支持`Pod`的扩容、缩容以及镜像版本的升级

```shell
kubectl create deployment [deployment名称] --image=[镜像] -n [命名空间] # 创建deployment，会自动创建相应的Pod
kubectl scale deployment [deployment名称] --replicas=2 -n [命名空间] # 将deployment下的pod扩容到2个
kubectl describe deployment [deployment名称] -n [命名空间] # 查看deployment的详细信息
kubectl delete deployment [deployment名称] -n [命名空间] # 删除deployment，级联删除其下的所有pod

kubectl apply -f deployment.yaml --record # 创建deployment，会根据yaml文件自动创建相应的Pod
kubectl get -f deployment.yaml # 查询Deployment
kubectl delete -f deployment.yaml # 删除deployment，级联删除其下的所有pod

kubectl set image deploy [deployment名称] [image名称]=[镜像版本] -n [命名空间] # 调整deployment下的Pod镜像版本
kubectl set image deploy tomcat tomcat=tomcat:jdk11

kubectl rollout undo deploy [deployment名称] --to-revision=1 -n [命名空间] # 回退deployment下Pod镜像的版本
kubectl rollout status deploy [deployment名称] -n [命名空间] # 回退状态
kubectl rollout history deploy [deployment名称] -n [命名空间] # 回退历史记录

# 基于现有Pod的CPU利用率选择要运行的Pod个数下限和上限
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

#### Service

`K8S `可以保证任意 `Pod` 挂掉时自动从任意节点启动一个新的`Pod`进行代替，以及某个`Pod`超负载时动态对`Pod`进行扩容。每当` Pod` 发生变化时其` IP`地址也会发生变化，且`Pod`只有在`K8S`集群内部才可以被访问，为了解决`Pod`发生变化导致其`IP`动态变化以及对外无法访问的问题，`K8S`引进了 `Service` 的概念

`K8S `使用 `Service `来管理同一组标签下的` Pod` ，当外界需要访问` Pod` 中的容器时，只需要访问 `Service` 的这个虚拟` IP` 和端口，由 `Service` 把外界的请求转发给它背后的`Pod`

```shell
kubectl get service -o wide -n [命名空间] # 查询service
# 创建service
kubectl create service nodeport [自定义名字] --tcp=80:80[service端口号:pod端口号] --node-port=31000[nodePort端口号]
```

`ClusterIP`：自动为当前`Service`分配虚拟`IP`，在不重启`Service`前提下该`IP`是不会改变的，**只能在集群内部访问**

`NodePort`：需要在`K8S`集群的所有 `Node` 节点上开放特定的端口【`30000-32767`】，通过（公网`ip` : 端口）访问`Service`服务。缺点：每个端口只能挂载一个`Service`

`service`创建成功后会通过`api-server`向`etcd`中写入相关配置信息，`kube-proxy`会监听创建好的配置信息并将最新的`service`配置信息转化成对应的访问规则，最终实现外网对`Pod`的访问。常见的`service`访问规则有：`iptables `和`ipvs`

`iptables`：`kube-proxy`为`service`后端的每个`Pod`创建对应的`iptables`规则，当用户访问`ClusterIP`时，直接将发送到`ClusterIP`的请求重定向到`Pod`

`ipvs`：`kube-proxy `监控`Pod`的变化并创建相应的`ipvs`规则，当用户访问`ClusterIP`时，直接将发送到`ClusterIP`的请求重定向到`Pod`，较复杂，需要单独安装

```shell
kubectl get pod -o wide -n [命名空间] | grep kube-proxy # 查询kube-proxy有关的pod
kubectl logs -n [命名空间] kube-proxy-7bst7 # 查询某一个kube的访问规则
kubectl edit configmap kube-proxy -n [命名空间] # 编辑kube-proxy的配置

# 部署应用并对外访问
kubectl create deployment [自定义名称] --image=[pod中容器镜像名称] --replicas=[pod副本数量] -n [命名空间名称]
kubectl expose deployment [自定义名称] --name=[service名称] --type=NodePort --port=[service端口] --target-port=[容器端口] -n [命名空间名称]
# 强制删除Pod
kubectl delete deployment [自定义名称] -n [命名空间名称] --force --grace-period=0
kubectl delete pod [pod名称] -n [命名空间名称] --force --grace-period=0
# 强制删除NameSpace
kubectl delete ns [命名空间名称] --force --grace-period=0 

# 外部服务访问
http://localhost:8080/api/v1/proxy/namespaces/<NAMESPACE>/services/<SERVICE-NAME>:<PORT-NAME>/
```

#### Ingress

`ingress `是`k8s`的资源对象，公开从集群外部到集群内服务的`HTTP`和`HTTPS`路由，用于定义请求如何转发到`Service`服务的规则。

`Ingress `暴露服务方式

- `Deployment`+`NodePort`的`Service`模式

  基于`Deployment`部署`ingress-controller`并创建对应的`Service`服务（`type`为`NodePort`），将`ingress`暴露在集群节点ip的特定端口上
  一般适用于宿主机ip地址相对固定不变的场景，由于`Nodeport`暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求，`NodePort`方式暴露 `ingress`虽然简单，但多了一层`NAT`，在请求量级很大时对性能会有一定影响

- `Deployment`+`LoadBalancer`的`Service`模式

  基于`Deployment`部署`ingress-controller`并创建对应的` Service `服务（`type`为`LoadBalancer`）
  此模式需要将`ingress`部署在公有云，大部分公有云都会为`LoadBalancer`的`service`自动创建一个负载均衡器，通常还绑定了公网地址，只要把域名解析指向该公网地址就实现了集群服务的对外暴露

- `DaemonSet`+`HostNetwork`+`nodeSelector`的`Service`模式

  基于`DaemonSet`结合`NodeSelector`来部署`ingress-controller`到特定的`node`上，再通过`HostNetwork`把该`pod`与宿主机`node`的网络打通，通过宿主机的`80/433`端口访问服务
  该方式相对`NodePort`模式的性能更好，缺点：由于直接利用宿主机节点的网络和端口，一个`node`只能部署一个`ingress-controller`，适合大并发的生产环境使用

![image-20230314141824393](C:\Users\Jerry\AppData\Roaming\Typora\typora-user-images\image-20230314141824393.png)

`Ingress`可为`Service`提供外部可访问的`URL`、负载均衡流量、终止`SSL/TLS`，以及基于名称的虚拟托管。 `Ingress`控制器通常负责通过负载均衡器来实现`Ingress`，尽管它也可以配置边缘路由器或其他前端来帮助处理流量。

### 探究与发现

#### 服务升级

kubernetes是如何友好的对服务进行版本升级的？

我们先创建一个`deployment`，对应维护3个`pod`

```shell
kubectl create deployment nginx --image=nginx --port=80 --replicas=3
```

如果某一天，我们发现当前版本存在一定问题，需要进行升级版本

```shell
kubectl set image deployment/nginx nginx=nginx:1.16.1
```

服务是如何升级的

```shell
kubectl describe deployments

Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  23m   deployment-controller  Scaled up replica set nginx-ff6774dc6 to 3
  Normal  ScalingReplicaSet  97s   deployment-controller  Scaled up replica set nginx-7f6dff8469 to 1 
  Normal  ScalingReplicaSet  93s   deployment-controller  Scaled down replica set nginx-ff6774dc6 to 2 from 3
  Normal  ScalingReplicaSet  93s   deployment-controller  Scaled up replica set nginx-7f6dff8469 to 2 from 1
  Normal  ScalingReplicaSet  90s   deployment-controller  Scaled down replica set nginx-ff6774dc6 to 1 from 2
  Normal  ScalingReplicaSet  90s   deployment-controller  Scaled up replica set nginx-7f6dff8469 to 3 from 2
  Normal  ScalingReplicaSet  87s   deployment-controller  Scaled down replica set nginx-ff6774dc6 to 0 from 1
```

可以看到，当第一次创建`Deployment`时，它创建了一个`ReplicaSet(nginx-ff6774dc6)` 并将其直接扩容至`3`个副本。更新`Deployment`时，它创建了一个新的`ReplicaSet(nginx-7f6dff8469)`，并将其扩容为`1`，等待其就绪；然后将旧`ReplicaSet`缩容到`12`， 将新的`ReplicaSet`扩容到`2`以便至少有`3`个 `Pod` 可用且最多创建`4`个`Pod`。 然后，它使用相同的滚动更新策略继续对新的`ReplicaSet`扩容并对旧的`ReplicaSet`缩容。 最后，你将有`3`个可用的副本在新的`ReplicaSet`中，旧`ReplicaSet`将缩容到`0`。

### yml文件编写

使用`ymal`文件对组件操作

```shell
kubectl apply -f [文件名].ymal # 使用文件去创建某个组件
kubectl get -f [文件名].ymal # 使用文件去查询某个组件
kubectl edit deployment/nginx-deployment # 更改deployment的配置
kubectl delete -f [文件名].ymal # 通过文件去删除某个组件
```

#### Pod文件定义

```shell
apiVersion: v1        　　          #必选，版本号，例如v1,版本号必须可以用 kubectl api-versions 查询到 .
kind: Pod       　　　　　　         #必选，Pod
metadata:       　　　　　　         #必选，元数据
  name: string        　　          #必选，Pod名称
  namespace: string     　　        #必选，Pod所属的命名空间,默认为"default"
  labels:       　　　　　　          #自定义标签
    - name: string      　          #自定义标签名字
  annotations:        　　                 #自定义注释列表
    - name: string
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像,否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]     　　        #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      　　             #容器的启动命令参数列表
    workingDir: string                     #容器的工作目录
    volumeMounts:     　　　　        #挂载到容器内部的存储卷配置
    - name: string      　　　        #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string                 #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean                 #是否为只读模式
    ports:        　　　　　　        #需要暴露的端口库号列表
    - name: string      　　　        #端口的名称
      containerPort: int                #容器需要监听的端口号
      hostPort: int     　　             #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string                  #端口协议，支持TCP和UDP，默认TCP
    env:        　　　　　　            #容器运行前需设置的环境变量列表
    - name: string      　　            #环境变量名称
      value: string     　　            #环境变量的值
    resources:        　　                #资源限制和请求的设置
      limits:       　　　　            #资源限制的设置
        cpu: string     　　            #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string                  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:       　　                #资源请求的设置
        cpu: string     　　            #Cpu请求，容器启动的初始可用数量
        memory: string                    #内存请求,容器启动的初始可用数量
    livenessProbe:      　　            #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject   　　    #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:     　　　　#Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false      　　    #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:        　　　　　　    #在该pod上定义共享存储卷列表
    - name: string     　　 　　    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}      　　　　    #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string      　　    #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　　　　    #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      　　　　            #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

#### Deployment文件定义

```shell
apiVersion: extensions/v1beta1
kind: Deployment
metadata: <Object>
spec: <Object>
  minReadySeconds: <integer> #设置pod准备就绪的最小秒数
  paused: <boolean> #表示部署已暂停并且deploy控制器不会处理该部署
  progressDeadlineSeconds: <integer>
  strategy: <Object> #将现有pod替换为新pod的部署策略
    rollingUpdate: <Object> #滚动更新配置参数，仅当类型为RollingUpdate
      maxSurge: <string> #滚动更新过程产生的最大pod数量，可以是个数，也可以是百分比
      maxUnavailable: <string> #
    type: <string> #部署类型，Recreate，RollingUpdate
  replicas: <integer> #pods的副本数量
  selector: <Object> #pod标签选择器，匹配pod标签，默认使用pods的标签
    matchLabels: <map[string]string> 
      key1: value1
      key2: value2
    matchExpressions: <[]Object>
      operator: <string> -required- #设定标签键与一组值的关系，In, NotIn, Exists and DoesNotExist
      key: <string> -required-
      values: <[]string>   
  revisionHistoryLimit: <integer> #设置保留的历史版本个数，默认是10
  rollbackTo: <Object> 
    revision: <integer> #设置回滚的版本，设置为0则回滚到上一个版本
  template: <Object> -required-
    metadata:
    spec:
      containers: <[]Object> #容器配置
      - name: <string> -required- #容器名、DNS_LABEL
        image: <string> #镜像
        imagePullPolicy: <string> #镜像拉取策略，Always、Never、IfNotPresent
        ports: <[]Object>
        - name: #定义端口名
          containerPort: #容器暴露的端口
          protocol: TCP #或UDP
        volumeMounts: <[]Object>
        - name: <string> -required- #设置卷名称
          mountPath: <string> -required- #设置需要挂载容器内的路径
          readOnly: <boolean> #设置是否只读
        livenessProbe: <Object> #就绪探测
          exec: 
            command: <[]string>
          httpGet:
            port: <string> -required-
            path: <string>
            host: <string>
            httpHeaders: <[]Object>
              name: <string> -required-
              value: <string> -required-
            scheme: <string> 
          initialDelaySeconds: <integer> #设置多少秒后开始探测
          failureThreshold: <integer> #设置连续探测多少次失败后，标记为失败，默认三次
          successThreshold: <integer> #设置失败后探测的最小连续成功次数，默认为1
          timeoutSeconds: <integer> #设置探测超时的秒数，默认1s
          periodSeconds: <integer> #设置执行探测的频率（以秒为单位），默认1s
          tcpSocket: <Object> #TCPSocket指定涉及TCP端口的操作
            port: <string> -required- #容器暴露的端口
            host: <string> #默认pod的IP
        readinessProbe: <Object> #同livenessProbe
        resources: <Object> #资源配置
          requests: <map[string]string> #最小资源配置
            memory: "1024Mi"
            cpu: "500m" #500m代表0.5CPU
          limits: <map[string]string> #最大资源配置
            memory:
            cpu:         
      volumes: <[]Object> #数据卷配置
      - name: <string> -required- #设置卷名称,与volumeMounts名称对应
        hostPath: <Object> #设置挂载宿主机路径
          path: <string> -required- 
          type: <string> #类型：DirectoryOrCreate、Directory、FileOrCreate、File、Socket、CharDevice、BlockDevice
      - name: nfs
        nfs: <Object> #设置NFS服务器
          server: <string> -required- #设置NFS服务器地址
          path: <string> -required- #设置NFS服务器路径
          readOnly: <boolean> #设置是否只读
      - name: configmap
        configMap: 
          name: <string> #configmap名称
          defaultMode: <integer> #权限设置0~0777，默认0664
          optional: <boolean> #指定是否必须定义configmap或其keys
          items: <[]Object>
          - key: <string> -required-
            path: <string> -required-
            mode: <integer>
      restartPolicy: <string> #重启策略，Always、OnFailure、Never
      nodeName: <string>
      nodeSelector: <map[string]string>
      imagePullSecrets: <[]Object>
      hostname: <string>
      hostPID: <boolean>
	  status: <Object>
```

#### Service文件定义

```shell
apiVersion: v1
kind: Service
matadata:                                #元数据
  name: string                           #service的名称
  namespace: string                      #命名空间
  labels:                                #自定义标签属性列表
    - name: string
  annotations:                           #自定义注解属性列表
    - name: string
spec:                                    #详细描述
  selector: []                           #label selector配置，将选择具有label标签的Pod作为管理 范围
  type: string                           #service的类型，指定service的访问方式，默认为clusterIp
  clusterIP: string                      #虚拟服务地址
  sessionAffinity: string                #是否支持session
  ports:                                 #service需要暴露的端口列表
  - name: string                         #端口名称
    protocol: string                     #端口协议，支持TCP和UDP，默认TCP
    port: int                            #服务监听的端口号
    targetPort: int                      #需要转发到后端Pod的端口号
    nodePort: int                        #当type = NodePort时，指定映射到物理机的端口号
  status:                                #当spce.type=LoadBalancer时，设置外部负载均衡器的地址
    loadBalancer:                        #外部负载均衡器
      ingress:                           #外部负载均衡器
        ip: string                       #外部负载均衡器的Ip地址值
        hostname: string                 #外部负载均衡器的主机名
```


