# 1.4.13 Namespace

Namespace（命名空间）是 Kubernetes 系统中另一个非常重要的概念，Namespace 在很多情况下用于实现多租房的资源隔离。Namespace 通过将集群内部的资源对象『分配』到不同的 Namespace 中，形成逻辑上分组的不同项目、小组或用户组，便于不同的分组在共享使用整个集群的资源的同时还能被分别管理。

Kubernetes 集群在启动后创建一个名为 default 的 Namespace，通过 kubectl 可以查看
```bash
kubectl get namespace
```
```text
NAME              STATUS   AGE
default           Active   15d
```

接下来，如果不特别指明 Namespace，则用户创建的 Pod、RC、Service 都将被系统创建到这个默认的名为 default 的 Namespace 中。

Namespace 的定义很简单
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

一旦创建了 Namespace，我们在创建资源对象时就可以指定这个资源对象属于哪个 Namespace。比如在下面的例子中定义了一个名为 busybox 的 Pod，并将其放入 development 这个 Namespace 里：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: development
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```

此时使用 kubectl get 命令查看，将无法显示，这是因为如果不加参数，默认显示的是 default 命名空间的资源对象，可以加入 `--namespace` 参数来查看具体命名空间的对象：
```bash
kubectl get pods --namespace=development
```
```text
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          2m28s
```

当给每个租房创建一个 Namespace 来实现多租房的资源隔离时，还能结合 Kubernetes 的资源配额管理，限定不同租房能占用的资源，例如 CPU 使用量、内存使用量等。关于资源配额管理的问题，在后的章节会详细介绍。
