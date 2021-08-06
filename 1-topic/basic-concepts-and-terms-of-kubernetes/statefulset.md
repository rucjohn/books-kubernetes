# StatefulSet

## 有状态的服务

在 Kubernetes 系统中，Pod 的管理对象 RC、Deployment、DeamonSet 和 Job都面向无状态的服务。但现实中有很多服务是有状态的，特别是一些复杂的中间件集群，例如 MySQL 集群、MongoDB 集群、Akka 集群、Zookeeper 集群等，这些应用集群有 4 个共同点：
* 每个节点起来都有固定的身份 ID，通过这个 ID，集群中的成员可以相互发现并通信。
* 集群的规模是比较固定的，集群规模不能随意变动。
* 集群中的每个节点都是有状态的，通常会持久化数据到永久存储中。
* 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损。

如果通过 RC 或 Deployment 控制 Pod 副本数量来实现上述有状态的集群，就会发现第 1 点就无法满足的，因为 Pod 的名称被随机产生，Pod 的 IP 地址也是在运行期才确定且可能有变动的，我们一中午无法为每个 Pod 都确定唯一不变的 ID。另外，为了能够在其他节点上恢复某个失败的节点，这种集群中的 Pod 需要挂接某种共享存储，为了解决这个问题， Kubernetes 从 1.4 版本开始引入了 PetSet 这个新的资源对象，并且在 1.5 版本时更名为 StatefulSet。

## 特性

StatefulSet 从本质上来说，可以看作 Deployment / RC 的一个特殊变种，它有如下特性：
* StatefulSet 里的每个 Pod 都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设 StatefulSet 的名称为 kafka，那么 Pod 叫 kafka-0、kafka-1、...，以此类推。
* StatefulSet 控制的 Pod 副本的启停顺序是受控的，操作第 n 个 Pod 时，前 n-1 个 Pod 已经是运行且准备好的状态。
* StatefulSet 里的 Pod 采用稳定的持久化存储卷，通过 PV 或 PVC 来实现，删除 Pod 时默认不会删除与 StatefulSet 相关的存储卷（为了保证数据的安全）。

## Headless Service

StatefulSet 除了要与 PV 卷捆绑使用以存储 Pod 的状态数据，还要与 Headless Service 配合使用，即在每个 StatefulSet 定义中都要声明它属于哪个 Headless Service。Headless Service 与普通的 Service 的关键区别在于，它没有 ClusterIP，如果解析 Headless Service 的 DNS 域名，则返回的是该 Service 对应的全部 Pod 的 Endpoint 列表。StatefulSet 在 Headless Service 的基础上又为 StatefulSet 控制的每个 Pod 实例都创建了一个 DNS 域名，这个域名的格式：
```text
$(podname).$(headless service name)
```

比如一个 3 节点 的 Kafka 的 StatefulSet 集群对应的 Headless Service 的名称为 kafka，StatefulSet 的名称为 kafka，则 StatefulSet 里的 3 个 Pod 的 DNS 名称分别为 kafka-0.kafka、kafka-1.kafka、kafka-2.kafka，这些 DNS 名称可以直接在集群的配置文件中固定下来。

