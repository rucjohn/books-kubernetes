# Node

除了 Master，Kubernetes 集群中的其他机器被称为 Node 节点，在较早的版本中也被称为 Minion。与 Master 一样，Node 节点可以是一台物理机，也可以是一台虚拟机。Node 节点才是 Kubernetes 集群中的工作负载节点，每个 Node 都会被 Master 分配一些工作负载（Docker 容器），当某个 Node 宕机时，其上的工作负载会被 Master 自动转移到其他节点上。

每个 Node 节点上都运行着以下一组关键进程：
* kubelet：负责 Pod 对应的容器的创建、启停等任务，同时与 Master 节点密切协作，实现集群管理的基本功能。
* kube-proxy：实现 Kubernetes Service 的通信与负载均衡机制的重要组件。
* Container Engine（docker）：容器引擎，负责本机的容器创建和管理工作。

Node 可以在运行期间动态增加到 Kubernetes 集群中，前提是这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下 kubelet 会向 Master 注册自己，这也是 Kubernetes 推荐的 Node 管理方式。一旦 Node 被纳入集群管理范围，kubelet 进程就会定时向 Master 汇报自身的情报，例如操作系统、Docker 版本、机器的 CPU 和内存情况，以及当前有哪些 Pod 在运行等，这样 Master 就可以获知每个 Node 的资源使用情况，并实现高效均衡的资源调度我策略。而某个 Node 在超过指定时间不上报信息时，会被 Master 判定为“失联”，Node 的状态被标记为不可用（Not Ready），随后 Master 会触发“工作负载大转移”的自动流程。

我们可以执行行下述命令查看在集群中有多少个 Node：
```bash
kubectl get nodes
```

输出

```text
NAME                         STATUS   ROLES    AGE   VERSION
prod-kubernetes-n1.fj.ascs   Ready    master   29h   v1.16.3
prod-kubernetes-n2.fj.ascs   Ready    worker   29h   v1.16.3
prod-kubernetes-n3.fj.ascs   Ready    worker   29h   v1.16.3
```

然后，通过 `kubectl describe node <NODE_NAME>` 查看某个 Node 的详细信息：
```bash
kubectl kubectl describe node prod-kubernetes-n1.fj.ascs
```

输出

```text
Name:               prod-kubernetes-n1.fj.ascs
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=prod-kubernetes-n1.fj.ascs
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 10.6.100.55/8
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.244.92.64
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 21 May 2020 10:01:49 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Fri, 22 May 2020 09:06:08 +0800   Fri, 22 May 2020 09:06:08 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Fri, 22 May 2020 15:07:52 +0800   Thu, 21 May 2020 10:01:46 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Fri, 22 May 2020 15:07:52 +0800   Thu, 21 May 2020 10:01:46 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Fri, 22 May 2020 15:07:52 +0800   Thu, 21 May 2020 10:01:46 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Fri, 22 May 2020 15:07:52 +0800   Thu, 21 May 2020 10:04:19 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.6.100.55
  Hostname:    prod-kubernetes-n1.fj.ascs
Capacity:
 cpu:                4
 ephemeral-storage:  49254916Ki
 hugepages-2Mi:      0
 memory:             16230472Ki
 pods:               110
Allocatable:
 cpu:                4
 ephemeral-storage:  45393330511
 hugepages-2Mi:      0
 memory:             16128072Ki
 pods:               110
System Info:
 Machine ID:                 fee4eb01de3947a99c756663d1afb1d1
 System UUID:                3160F341-2249-4ED1-ABF4-F1A25406FFED
 Boot ID:                    c2af4fc6-76f8-474b-b415-ef4da1e5473f
 Kernel Version:             3.10.0-957.el7.x86_64
 OS Image:                   CentOS Linux 7 (Core)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://19.3.9
 Kubelet Version:            v1.16.3
 Kube-Proxy Version:         v1.16.3
PodCIDR:                     10.244.0.0/24
PodCIDRs:                    10.244.0.0/24
Non-terminated Pods:         (11 in total)
  Namespace                  Name                                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                                  ------------  ----------  ---------------  -------------  ---
  default                    demo-789cc6d8b9-cbnhh                                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         4h6m
  kube-system                calico-kube-controllers-6b64bcd855-tb7fx              0 (0%)        0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                calico-node-62blr                                     250m (6%)     0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                coredns-67c766df46-gw66r                              100m (2%)     0 (0%)      70Mi (0%)        170Mi (1%)     29h
  kube-system                coredns-67c766df46-htzxj                              100m (2%)     0 (0%)      70Mi (0%)        170Mi (1%)     29h
  kube-system                etcd-prod-kubernetes-n1.fj.ascs                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                kube-apiserver-prod-kubernetes-n1.fj.ascs             250m (6%)     0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                kube-controller-manager-prod-kubernetes-n1.fj.ascs    200m (5%)     0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                kube-proxy-fwwmt                                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         29h
  kube-system                kube-scheduler-prod-kubernetes-n1.fj.ascs             100m (2%)     0 (0%)      0 (0%)           0 (0%)         29h
  traefik-ingress            traefik-f44gl                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         22h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                1 (25%)     0 (0%)
  memory             140Mi (0%)  340Mi (2%)
  ephemeral-storage  0 (0%)      0 (0%)
Events:              <none>
```

上述命令展示了 Node 的如下关键信息：
* Node 的基本信息：名称、标签、创建时间等。
* Node 当前的运行状态：Node 启动后做一系列的自检工作，比如磁盘空间是否不足（DiskPressure）、内存是否不足（MemoryPressure）、网络是否正常（NetworkUnavailable）、PID 资源是否充足（PIDPressure）。在一切正常时设置 Node 为 Ready 状态（Ready=True），该状态表示 Node 处于健康状态，Master 将可以在其上调度新的任务（启动 Pod）。
* Node 的主机地址和主机名。
* Node 上的资源数量：描述 Node 可用的系统资源，包括 CPU、内存、最大调度 Pod 数量等。
* Node 可分配的资源数量：描述 Node 当前可用于分配的资源数量。
* 主机系统信息：包括主机 ID、系统 UUID、Linux kernel 版本号、操作系统类型与版本、Docker 版本、kubelet 与 kube-proxy 版本等。
* 当前运行的 Pod 列表概要信息。
* 已分配的资源使用概要信息，例如资源申请的最低、最大允许使用量占系统总量的百分比。
* Node 相关的 Event 信息。
