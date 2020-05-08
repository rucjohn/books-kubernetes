# 1.4.6 Deployment

Deployment 是 Kubernetes 在 1.2 版本中引入的新概念，用于更好地解决 Pod 的编排问题。为此，Deployment 在内部使用了 Replica Set 来实现目的，无论从 Deployment 的作用与目的、YAML 定义，还是从它的具体命令行操作来看，我们都可以把它看作 RC 的一次升级，两者的相似度越过 90%。

Deployment 相对于 RC 的一个最大的升级是我们可以随时知道当前 Pod 『部署』的进度。实际上由于一个 Pod 的创建、调度、绑定节点及在目标 Node 上启动对应的容器这一完整过程需要一定的时间，所以我们期待系统启动 N 个 Pod 副本的目标状态，实际上是一个连续变化的『部署过程』导致的最终状态。

## 应用场景

Deployment 的典型使用场景有以下几个：
* 创建一个 Deployment 对象来生成对应的 Replica Set 并完成 Pod 副本的创建。
* 检查 Deployment 的状态来看部署动作是否完成（Pod 副本数量是否达到预期的值）。
* 更新 Deployment 以创建新的 Pod（比如镜像升级）。
* 如果当前 Deployment 不稳定，则回滚到一个早先的 Deployment 版本。
* 暂停 Deployment 以便于一次性修改多个 PodTemplateSpec 的配置项，之后再恢复 Deployment，进行新的发布。
* 扩展 Deployment 以应对高负载。
* 查看 Deployment 的状态，以此作为发布是否成功的指标。
* 清理不再需要的旧版本 ReplicaSets。

除了 API 声明与 Kind 类型等有所区别，Deployment 的定义与 Replica Set 的定义很相似：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
```

## 示例

下面通过运行一些例子来直观地感受 Deployment 的概念。创建一个名为 tomcat-deployment.yaml 的 Deployment 描述文件，内容如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec: 
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        image: tomcat:8.0.35
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
```

创建 Deployment
```bash
kubectl create -f tomcat-deployment.yaml
```

查看 Deployment 信息
```bash
kubectl get deployments
```
```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
frontend           1/1     1            1           4m22s
```

上述输出解释如下：
* READY：当前 Replica 值 / Pod 副本数量的期望值。实际上当前 Replica 值在 Deployment 创建过程中不断增加，直到达到期望值为止，表明整个部署过程中完成。
* UP-TO-DATE：最新版本的 Pod 的副本数量，用于指示在滚动升级的过程中，有多少 Pod 副本已经成功升级。
* AVAILABLE:：当前集群中可用的 Pod 副本数量，即集群中当前存活的 Pod 数量。

## 命名对应关系

查看对应的 Replica Set，我们看到它的命名与 Deployment 的名称有关系：
```bash
kubectl get rs
```
```text
NAME                          DESIRED   CURRENT   READY   AGE
frontend-b488bcf84            1         1         1       11m
```

查看创建的 Pod，我们发现 Pod 命名以 Deployment 对应的 Replica Set 的名称为前缀，这种命名很清晰地表明了一个 Replica Set 创建了哪些 Pod，对于 Pod 滚动升级这种复杂的过程来说，很容易排查错误：
```bash
kubectl get pods
```
```text
NAME                                READY   STATUS        RESTARTS   AGE
frontend-b488bcf84-s7q9s            1/1     Running       0          14m
```

运行 `kubectl describe deployments <DEPLOYMENT_NAME>` 可以清楚地看到 deployment 控制的 Pod 的水平扩展过程，具体内容可参见第 3 章的说明。

## ......

Pod 的管理对象，除了 RC 和 Deployment ，还包括 ReplicaSet、DaemonSet、StatefulSet、Job等，分别用于不同的应用场景中，将在第 3 章进行详细介绍。
