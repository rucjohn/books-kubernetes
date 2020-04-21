# 1.4 Kubernetes 的基本概念和术语

Kubernetes 中的大部分概念如 Node、Pod、Replication Controller、Service 等都可以被看作一种资源对象，几乎所有资源对象都可以通过 Kubernetes 提供的 kubectl 工具（或者 API 编程调用）执行增、删、改、查等操作并将其保存在 etcd 中持久化存储。从这个角度来看，Kubernetes 其实是一个高度自动化的资源控制系统，它通过跟踪对比 etcd 库里保存的『资源期望状态』与当前环境中的『实际资源状态』的差异来实现自动控制和自动纠错的高级功能。

在声明一个 Kubernetes 资源对象的时候，需要注意一个关键属性：apiVersion。以下面的 Pod 声明为例，可以看到 Pod 这种资源对象归属于 v1 这个核心 API。
```
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  labels:
    name: myweb
spec:
  containers:
  - name: myweb
    image: kubeguide/tomcat-app:v1
    ports:
      - containerPort: 8080
```

Kubernetes 平台采用了『核心 + 外围扩展』的设计思路，在保持平台核心稳定的同时具备持续演进升级的优势。Kubernetes 大部分觉的核心资源对象都归属于 v1 这个核心 API。比如 Node、Pod、Service、Endpoints、Namespace、RC、PersistentVolume 等。在版本迭代过程中，Kubernetes 先后扩展了 extensions/v1beta1、apps/v1beta1、apps/v1beta2 等 API 组，而在 1.9 版本之后引入了 apps/v1 这个正式的扩展 API 组，正式淘汰了 extensions/v1beta1、apps/v1beta1、apps/v1beta2 这三个 API 组。

我们可以采用 YAML 或 JSON 格式声明（定义或创建）一个 Kubernetes 资源对象，每个资源对象都有自己的特定语法格式（可以理解为数据库中一个特定的表），但随着 Kubernetes 版本的持续升级，一些资源对象会不断引入新的属性。为了在不影响当前功能的情况下引入特性的支持，我们通常会采用下面两种典型方法：
- 方法 1，在设计数据库表的时候，在每个表中都增加一个很长的备注字段，之后扩展的数据以某种格式（如 XML、JSON、简单字符串拼接等）放入备注字段。因为数据库表的结构没有发生变化，所以此时程序的改动范围是最小的，风险也更小，但看越来不太美观。
- 方法 2，直接修改数据库表，增加一个或多个新的列，此时程序的改动范围较大，风险更大，但看来比较美观。

显然，两种方法都不完美。更加优雅的做法是，先采用方法 1实现这个新特性，经过几个版本迭代，等新特性变得稳定成熟了以后，可以在后续版本中采用方法 2升级到正式版。为此，Kubernetes 为每个资源对象都增加了类似数据库表里备注字段的通用属性 Annotations，以实现方法 1 的升级。以 Kubernetes 1.3 版本引入的 Pod 的 Init Container 新特性为例，一开始，Init Container 的定义是在 Annotations 中声明的，如下面代码中部分所示，是不是很不美观？
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    name: myapp
  annotations:
    pod.beta.kubernetes.ip/init-containers: '[
      {
        "name": "init-mydb",
        "image": "busybox",
        "command": [......]
      }
    ]'
spec:
  containers:
  ......
```

在 Kubernetes 1.8 版本以后，Init Container 特性完全成熟，其定义被放入 Pod 的 spec.initContainers 一节，看起来优雅了很多：
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    name: myapp
spec:
  initContainers:
  - name: init-mydb
    image: busybox
    command:
    - xxx
```

在 Kubernetes 1.8 中，资源对象中的很多 Alpha、Beta 版本的 Annotations 被取消，升级成了常规定义方式，在学习 Kubernetes 的过程中需要特别注意。

下面介绍 Kubernetes 中重要的资源对象。

