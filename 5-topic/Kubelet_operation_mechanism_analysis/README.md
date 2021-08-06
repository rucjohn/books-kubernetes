# Kubelet 运行机制解析

在 Kubernetes 集群中，在每个 Node（早期称 Minion）上都会启动一个 kubelet 服务里程。该进程用于处理 Master 下发到本节点的任务，管理 Pod 及 Pod 中的容器。每个 kubelet 进程都会在 apiserver 上注册节点自身的信息，定期向 Master 汇报 节点的资源的使用情况，并通过 cAdvisor 监控容器和节点资源。
