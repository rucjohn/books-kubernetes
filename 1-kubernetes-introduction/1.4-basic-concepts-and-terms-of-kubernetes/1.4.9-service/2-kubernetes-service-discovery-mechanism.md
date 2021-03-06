# 2. Kubernetes 的服务发现机制

任何分布式系统都会涉及『服务发现』这个基础问题，大部分分布式系统都通过提供特定的 API 接口来实现服务发现功能，但这样做会导致平台的侵入性比较强，也增加了开发、测试的难度。Kubernetes 则采用了直观朴素的思路去解决这个棘手的问题。

首先，每个 Kubernetes 中的 Service 都有唯一的 ClusterIP 及唯一的名称，而名称是由开发者自己定义的，部署时也没必要改变，所以完全可以被固定在配置中。接下来的问题就是如何通过 Service 的名称找到对应的 ClusterIP。

最早时 Kubernetes 采用了 Linux 环境变量解决这个问题，即每个 Service 都生成一些对应的 Linux 环境变量（ENV），并在每个 Pod 的容器启动时自动注入这些环境变量。考虑到通过环境变量获取 Service 地址的方式仍然不太方便、不够直观，后来 Kubernetes 通过 Add-On 增值包引入 DNS 系统，把服务名作为 DNS 域名，这样程序就可以直接使用服务名建立通信连接了。目前 Kubernetes 上的大部分应用都已经采用了 DNS 这种新兴的服务发现机制，后面会讲解如何部署 DNS 系统。

