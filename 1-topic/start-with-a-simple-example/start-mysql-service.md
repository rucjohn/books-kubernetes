# 启动 MySQL 服务

## 创建 RC

首先，为 MySQL 服务创建一个 RC 定义文件`mysql-rc.yaml`，下面给出的该文件的完整内容和解释：

```yaml
apiVersion: v1
kind: ReplicationController             # 副本控制器RC
metadata:
  name: mysql                           # RC的名称，全局唯一
spec:
  replicas: 1                           # Pod副本的期待数量
  selector:
    app: mysql                          # 符合目标的Pod拥有此标签
  template:                             # 根据此模板创建Pod的副本（实例）
    metadata:
      labels:
        app: mysql                      # Pod副本拥有的标签，对应RC的selector
    spec:
      containers:                       # Pod内容器的定义部分
      - name: mysql                     # 容器的名称
        image: mysql:5.7                # 容器对应的Docker Image
        ports:
        - containerPort: 3306           # 容器应用监听的端口号
        env:                            # 注入容器内的环境变量
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```

> 由于 mysql:latest（>5.7）需要认证，所以本例中采用 mysql 5.7 版本进行实验

以上 YAML 文件中的 kind 属性用来表明此资源对象的类型，比如这里的值为 ReplicatonController，表示这是一个RC；在 spec 一节中是 RC 的相关属性定义，比如 spec.selector 是RC 的标签选择器，即监控和管理拥有这些标签的 Pod 实例，确保在当前集群始终有且仅有 replicas 个 Pod 实例，这里设置 replicas=1，表示只能运行一个 MySQL Pod 实例。当在集群中运行的 Pod 数量少于 replicas 时，RC 会根据在 spec.template 一节中定义的 Pod 模板来生成一个新的 Pod 实例，spec.template.metadata.labels指定了该 Pod 的标签，需要特别注意的是：这里的 labels 必须匹配之前的 spec.selector，否则此 RC 每创建一个无法匹配 Label 的 Pod，就会不停地尝试创建新的 Pod，陷入恶性循环中。

## 发布

在创建好 `mysql-rc.yaml` 文件后，为了将它发布到 Kubernetes 集群中：

```bash
kubectl create -f mysql-rc.yaml
kubectl get rc
```

结果输出

```text
NAME    DESIRED   CURRENT   READY   AGE
mysql   1         1         1       47h
```

查看 Pod 的创建情况时，可以运行下面的命令：

```bash
kubectl get pods
```

结果输出

```text
NAME          READY   STATUS    RESTARTS   AGE
mysql-h77kb   1/1     Running   0          47h
```

我们看到一个名为 `mysql-xxxxx` 的Pod实例，这是 Kubernetes 根据 mysql 这个RC 的定义自动创建的 Pod。由于 Pod 的调度和创建需要花费一定的时间，比如需要一定的时间来确定调度到哪个节点上，以及下载 Pod 里的容器镜像需要一段时间，所以我们一开始看到 Pod 状态显示为 Pending。在 Pod 成功创建完成以后，状态最终会被更新为 Running。

我们通过 docker ps 指令查看正在运行的容器，发现提供 MySQL 服务的 Pod 容器已经创建并正常运行了，此外会发现 MySQL Pod 对应的容器还多创建了一个来自谷歌的 pause 容器，这就是 Pod 的 『根容器』，详见 1.4.3 节的说明。

```bash
docker ps | grep mysql
```

结果输出
```text
6140e316610e        mysql                  "docker-entrypoint.s…"   2 hours ago         Up 2 hours                              k8s_mysql_mysql-vgzw4_default_d473f994-35b9-4d36-bf37-0be6265d3ce5_0
5a5b88a0db5d        k8s.gcr.io/pause:3.2   "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_mysql-vgzw4_default_d473f994-35b9-4d36-bf37-0be6265d3ce5_0
```

## 创建 Service

最后，创建一个与之关联的 Kubernetes Server --- MySQL 的定义文件（文件名为 mysql-svc.yaml），完整的内容和解释如下：

```yaml
apiVersion: v1
kind: Service           # 表明是 Kubernetes Service
metadata:
  name: mysql           # Service 的全局唯一名称
spec:
  ports: 
    - port: 3306        # Service 提供服务的端口号
  selector:             # Service 对应的 Pod 拥有这里定义的标签
    app: mysql
```

其中，metadata.name 是 Service 的服务名（ServiceName）；port 属性则定义了 Service 的虚端口；spec.selector 确定了哪些 Pod 副本（实例）对应本服务。类似地，我们通过 kubectl create 创建 Service 对象。

创建 Service：

```bash
kubectl create -f mysql-svc.yaml
kubectl get svc
```

结果输出

```text
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    24h
mysql        ClusterIP   10.111.135.20   <none>        3306/TCP   6s
```

可以发现，MySQL 服务被分配了一个值为 `10.111.135.20` 的 Cluster IP。随后，Kubernetes 集群中其他新创建的 Pod 就可以通过 Service 的 Cluster IP + 端口号 3306 来连接和访问它了。

通常， Cluster IP 是在 Service 创建后由 Kubernetes 系统自动分配的，其他的 Pod 无法预先知道某个 Service 的 Cluster IP 地址，因此需要一个服务发现机制来找到这个服务。为此，最初时，Kubernetes 巧妙地使用了 Linux 环境变量（Environment Variable）来解决这个问题，后面会详细说明其机制。现在只需知道，根据 Service 的唯一名称，容器可以从环境变量中获取 Service 对应的 Cluster IP 地址和端口，从而发超 TCP/IP 连接请求。

