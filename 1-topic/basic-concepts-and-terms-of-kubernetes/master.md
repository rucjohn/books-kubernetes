# Master

Kubernetes 里的 Master 指的是集群控制节点，在每个 Kubernetes 集群里都需要有一个 Master 来负责整个集群的管理和控制，基本上 Kubernetes 的所有控制命令都发给它，它负责具体的执行过程，我们后面执行的所有命令基本都是在 Master 上运行的。Master 通常会占据一个独立的服务器（高可用部署建议用 3 台服务器），主要原因是它太重要了，是整个集群的『首脑』，如果它宕机或者不可用，那么对集群内应用的管理都将失效。

在 Master 上运行着以下关键进程：

* Kubernetes API Server（kube-apiserver）：提供了 HTTP Rest 接口的关键服务进程，是 Kubernetes 里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
* Kubernetes Controller Manager（kube-controller-manager）：Kubernetes 里所有资源对象的自动化控制中心，可以将其理解为资源对象的『大总管』。
* Kubernetes Scheduler（kube-scheduler）：负责资源调度（Pod 调度）的进程，相当于公交公司的『调度室』。

另外，在 Master 上通常还需要部署 etcd 服务，因为 Kubernetes 里所有资源对象的数据都保存在 etcd 中。

