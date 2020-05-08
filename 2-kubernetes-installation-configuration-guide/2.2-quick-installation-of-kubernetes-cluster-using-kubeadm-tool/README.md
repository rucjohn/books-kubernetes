# 2.2 使用 kubeadm 工具快速安装 Kubernetes 集群

最简单的方法是使用 `yum install kubernetes` 命令安装 Kubernetes 集群，但仍需修改各组件的启动参数，才能完成对 Kubernetes 集群的配置，整个过程比较复杂，也容易出错。Kubernetes 从 1.4 版本开始引入了命令行工具 kubeadm ，致力于简化集群的安装过程，并解决 Kubernetes 集群的高可用问题。在 Kubernetes 1.13 版本中，kubeadm 工具进入 GA 阶段，宣称已经为生产环境应用准备就绪。

本节先讲解基于 kubeadm 的安装过程（以 CentOS 7 为例）。

