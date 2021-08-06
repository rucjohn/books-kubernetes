# Service

## 概述

Service 服务也是 Kubernetes 里的核心资源对象之一，Kubernetes 里的每个 Service 其实就是我们经常提供的微服务架构中的一个微服务，之前讲解 Pod、RC 等资源对象其实都是在为讲解 Kubernetes Service 使用铺垫。下图显示了 Pod、RC 与 Service 的逻辑关系：

![](../../../gitbook/assets/topic_1/1-12.jpg)

上图可以看到，Kubernetes 中的 Service 定义了一个服务的访问入口地址，前端的应用（Pod）通过这个入口地址访问其背后的一组由 Pod 副本组成的集群实例，Service 与其后端 Pod 副本集群之间通过 Label Selector 来实现无缝对接。RC 的作用实际上是保证 Service 的服务能力和服务质量始终符合预期标准。

通过分析，识别并建模系统中的所有服务为微服务 --- Kubernetes Service，我们的系统最终由多个提供不同业务能力而又彼此独立的微服务单元组成的，服务之间通过 TCP/IP 进行通信，从而形成了强大而又灵活的弹性网络，拥有强大的分布式能力、弹性扩展能力、容错能力，程序架构也变得简单和直观许多，如下图所示：

![](../../../gitbook/assets/topic_1/1-13.jpg)

既然每个 Pod 都会被分配一个单独的 IP 地址，而且每个 Pod 都提供了一个独立的 Endpoint（Pod IP + ContainerPort）以被客户端访问，现在多个 Pod 副本组成了一个集群来提供服务，那么客户端如何来访问它们呢？一般的做法是部署一个负载均衡器（软件或硬件），为这组 Pod 开启一个对外的服务端口（如 8080），并且将这些 Pod 的 Endpoint 列表加入服务端口的转发列表，客户端就可以通过负载均衡器的对外 IP 地址 + 服务端口来访问此服务。客户端的请求最后会被转发到哪个 Pod，由负载均衡器的算法所决定。

Kubernetes 也遵循上述的常规做法，运行在每个 Node 上的 kube-proxy 进程其实就是一个智能的软件负载均衡器，负责把对 Service 的请求转发到后端的某个 Pod 实例上，并在内部实现服务的负载均衡与会话保持机制。但 Kubernetes 发明了一种很巧妙又影响深远的设计：Service 没有共用一个负载均衡器的 IP 地址，每个 Service 都被分配了一个全局唯一的虚拟 IP 地址，这个 虚拟 IP 被称为 ClusterIP。这样一来，每个服务就变成了具备唯一 IP 地址的通信节点，服务调用就变成了最基础的 TCP 网络通信问题。

我们知道，Pod 的 Endpoint 地址会随着 Pod 的销毁和重新创建而发生改变，因为新的 Pod 的 IP 地址与之前旧的 Pod 的不同。而 Service 一旦被创建，Kubernetes 就会自动为它分配一个可用的 ClusterIP，而且在 Service 的整个生命周期内，它的 ClusterIP 不会发生改变。于是，服务发现这个棘手的问题在 Kubernetes 的架构 里也得以轻松解决：只要用 Service 的 Name 与 Service 的 ClusterIP 地址做一个 DNS 域名映射 即可完美解决问题。现在想想，这真是一个很棒的设计。

### 示例

创建一个名为 tomcat-service.yaml 的定义文件：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 8080
  selector:
    tier: frontend
```

上述内容定义了一个名为 tomcat-service 的 Service，它的服务端口为 8080，拥有 "tier=frontend" Label 的所有 Pod 实例都属于它，运行下面的命令进行创建：
```bash
kubectl create -f tomcat-service.yaml
```

我们之前在 tomcat-deployment.yaml 里定义的 Tomcat 的 Pod 刚好拥有这个标签，所以刚才创建的 tomcat-service 已经对应一个 Pod 实例，运行下面的命令可以查看 tomcat-service 的 Endpoint 列表
* `10.244.2.13` 是 Pod IP 地址
* `8080` 是 Container 暴露的端口

```bash
kubectl get endpoints
```
```text
NAME             ENDPOINTS                         AGE
kubernetes       10.6.100.56:6443                  14d
tomcat-service   10.244.2.13:8080                  20s
```

但是，说好的 Service 的 ClusterIP呢？运行下面的命令，可以看到 Service 被分配的 ClusterIP 及更多的信息：
```bash
kubectl get svc tomcat-service -o yaml
```
```text
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-05-07T04:01:35Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:ports:
          .: {}
          k:{"port":8080,"protocol":"TCP"}:
            .: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector:
          .: {}
          f:tier: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: kubectl
    operation: Update
    time: "2020-05-07T04:01:35Z"
  name: tomcat-service
  namespace: default
  resourceVersion: "1839657"
  selfLink: /api/v1/namespaces/default/services/tomcat-service
  uid: 276eb5f8-4335-477e-bb0e-18e97e182935
spec:
  clusterIP: 10.98.133.184
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    tier: frontend
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

解释如下：
* targetPort 属性用来确定提供该服务的容器所暴露（EXPOSE）的端口号，即具体业务进程在容器内的 targetPort 上提供的 TCP/IP 接入；
* port 属性则定义了 Service 的虚端口。
* 前面定义 Tomcat 服务时没有指定 targetPort，则默认 targetPort 与 Port 相同。

### 多端口

接下来看看 Service 多端口问题

很多服务都存在多个端口的问题，通常一个端口提供业务服务，另外一个端口提供管理服务，比如 mycat、codis 等常见中间件。Kubernetes Service 支持多个 Endpoint，在存在多个 Endpoint 的情况下，要求每个 Endpoint 都定义一个名称来区分。下面是 Tomcat 多端口 Service 定义样例：
```
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 8080
    name: service-port
  - port: 8005
    name: shutdown-port
  selector:
    tier: frontend
```

多端口为什么需要给每个端口都命令呢？这就涉及 Kubernetes 的服务发现机制了，接下来进行讲解。

## Kubernetes 的服务发现机制

任何分布式系统都会涉及『服务发现』这个基础问题，大部分分布式系统都通过提供特定的 API 接口来实现服务发现功能，但这样做会导致平台的侵入性比较强，也增加了开发、测试的难度。Kubernetes 则采用了直观朴素的思路去解决这个棘手的问题。

首先，每个 Kubernetes 中的 Service 都有唯一的 ClusterIP 及唯一的名称，而名称是由开发者自己定义的，部署时也没必要改变，所以完全可以被固定在配置中。接下来的问题就是如何通过 Service 的名称找到对应的 ClusterIP。

最早时 Kubernetes 采用了 Linux 环境变量解决这个问题，即每个 Service 都生成一些对应的 Linux 环境变量（ENV），并在每个 Pod 的容器启动时自动注入这些环境变量。考虑到通过环境变量获取 Service 地址的方式仍然不太方便、不够直观，后来 Kubernetes 通过 Add-On 增值包引入 DNS 系统，把服务名作为 DNS 域名，这样程序就可以直接使用服务名建立通信连接了。目前 Kubernetes 上的大部分应用都已经采用了 DNS 这种新兴的服务发现机制，后面会讲解如何部署 DNS 系统。

## 外部系统访问 Service 的问题

### IP

为了更深入地理解和掌握 Kubernetes，我们需要弄明白 Kubernetes 里的 3 种 IP：
* Node IP：Node 的 IP 地址
* Pod IP：Pod 的 IP 地址
* Cluster IP：Service 的 IP 地址

首先，Node IP 是 Kubernetes 集群中每个节点的物理网卡的 IP 地址，是一个真实存在的物理网络，所有属于这个网络的服务器都能通过这个网络直接通信，不管其中是否有部分节点不属于这个 Kubernetes 集群。这也表明在 Kubernetes 集群之外的节点访问 Kubernetes 集群之内的某个节点或者 TCP/IP 服务时，都必须通过 Node IP 通信。

其次，Pod IP 是每个 Pod 的 IP 地址，它是 Docker Engine 根据 docker0 网桥的 IP 地址段进行分配的，通常是一个虚拟的二层网络，前面说过，Kubernetes 要求位于不同 Node 上的 Pod 都能够直接彼此通信，所以 Kubernetes 里一个 Pod 里的容器访问另一个 Pod 里的容器时，就是通过 Pod IP所在的虚拟二层网络进行通信的，而真实的 TCP/IP 流量是通过 Node IP所在的物理网卡流出的。

最后，说说 Service 的 Cluster IP，它也是一种虚拟的 IP，但更像一个『伪造』的 IP 网络，原因如下：
* ClusterIP 仅仅作用于 Kubernetes Service 这个对象，并由 Kubernetes 管理和分配 IP 地址（来源于 ClusterIP 地址池）。
* ClusterIP 无法被 ping，因为没有一个『实体网络对象』来响应。
* ClusterIP 只能结合 Service Port 组成一个具体的通信端口，单独的 ClusterIP 不具备 TCP/IP 通信的基础，并且它们属于 Kubernetes 集群这样一个封闭的空间，集群外的节点如果要访问这个通信端口，则需要做一些额外的工作。
* 在 Kubernetes 集群内，Node IP 网、Pod IP 网和 Cluster IP 网之间的通信，采用的是 Kubernetes 自己设计的一种编程方式的特殊路由规则，与我们熟知的 IP 路由有很大的不同。

### NodePort

根据上面的分析和总结，我们基本明白了：Service 和 Cluster IP 属于 Kubernetes 集群内部的地址，无法在集群外部直接使用这个地址。那么矛盾来了：实际上在我们开发的业务系统中肯定有一部分服务要提供给 Kubernetes 集群外部的应用或用户来使用，典型的例子就是 Web 端口的服务模块，比如上面的 tomcat-service，那么用户怎么访问它？

采用 NodePort 是解决上述问题的最直接、有效的常见做法。以 tomcat-service 为例，在 Service 的定义里如下扩展即可：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 31002
  selector:
    tier: frontend
```

其中，nodePort:31002 这个属性表明手动指定 tomcat-service 的 NodePort 为 31002，否则 Kubernetes 会自动分配一个可用的端口。接下来在浏览器里访问 http://<NODE_IP>:31002，就可以看到 tomcat 的欢迎界面。

![](../../../gitbook/assets/topic_1/1-14.jpg)

NodePort 的实现方式是在 Kubernetes 集群里的每个 Node 上都为需要外部访问的 Service 开启一个对应的 TCP 监听端口，外部系统只要用任意一个 NodeIP + NodePort 即可访问此服务，在任意 Node 上运行 netstat 命令，就可以看到有 NodePort 被监听：
```bash
netstat -ntpl | grep 31002
```
```text
tcp        0      0 0.0.0.0:31002           0.0.0.0:*               LISTEN      178045/kube-proxy
```

### Load balancer

但 NodePort 还没有完全解决外部访问 Service 的所有问题，比如负载均衡问题。假如在我们的集群中有 10 个 Node，则此时最好有一个负载均衡器，外部的请求只需要访问此负载均衡器的 IP 地址，由负载均衡器负载转发流量到后面某个 Node 的 NodePort 上，如下图：

![](../../../gitbook/assets/topic_1/1-15.jpg)

上图 Load balancer 组件独立于 Kubernetes 集群之外，通常是由一个硬件的负载均衡器，或者是以软件方式实现的，例如 HAProxy 或 Nginx。对于每个 Service，我们通常需要配置一个对应的 Load balancer 实例来转发流量到后端的 Node 上，这的确增加了工作量及出错的概率。于是 Kubernetes 提供了自动化的解决方案，如果我们的集群运行在谷歌的公有云 GCE 上，那么只要把 Service 的 type=NodePort 改为 type=LoadBalancer，Kubernetes 就会自动创建一个对应的 Load balancer 实例并返回它的 IP 地址提供外部客户端使用。其他公有云只要实现了支持此特性的驱动，则也可以达到上述目的。此外裸机上的类似机制（Bare Metra Service Load Balancers）也在被开发。
