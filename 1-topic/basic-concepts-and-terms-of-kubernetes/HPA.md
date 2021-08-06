# Horizontal Pod Autoscaler

通过手工执行 `kubectl scale` 命令，我们可以实现 Pod 扩容或缩容。如果仅仅到此为止，显然不符合谷歌对 Kubernetes 的定位目标 --- 自动化、智能化。在谷歌看来，分布式系统要能够根据当前负载的变化自动触发水平扩容或缩容，因为这一过程可能是频繁发生的、不可预料的，所以手动控制的方式是不现实的。

因此，在 Kubernetes 1.0 版本实现后，就有人在默默研究 Pod 智能扩容的特性了，并在 Kubernetes 1.1 中首次发布重量级新特性 --- Horizontal Pod Autocaling（Pod 横向自动扩容，HPA）。在 Kubernetes 1.2 中 HPA 被升级为稳定版本（apiVersion: autoscaling/v1），但仍然保留了旧版本（apiVersion: extensions/v1beta1）。Kubernetes 从 1.6 版本开始，增强了根据应用自定义的指标进行自动扩容和缩容的功能，API 版本为 autoscaling/v2alpha1，并不断演进。

## 原理

HPA 与之前的 RC、Deployment 一样，也属于一种 Kubernetes 资源对象。通过跟踪分析指定 RC 控制的所有目标 Pod 的负载变化情况，来确定是否需要有针对性地调整目标 Pod 的副本数量，这就是 HPA 的实现原理。

## 度量指标

当前 HPA 有以下两种方式作为 Pod 负载的度量指标：
* CPUUtilizationPercentage
* 应用程序自定义的度量指标，比如服务在每秒内的相应请求数（TPS 或 QPS）

CPUUtilizationPercentage 是一个算术平均值，即目标 Pod 所有副本自身的 CPU 利用率的平均值。一个 Pod 自身的 CPU 利用率是该 Pod 当前 CPU 除以 它的 Pod Request 的值，比如定义一个Pod Pod Requests 为 0.4，而当前的 Pod CPU 使用量为 0.2，则它的 CPU 使用率为 50%，这样就可以算出一个 RC 控制的所有 Pod 副本的 CPU 利用率的算术平均值了。如果某一时刻 CPUUtilizationPercentage 的值超过 80%，则意味着当前 Pod 副本数量可能不足以支撑接下来更多的请求，需要进行动态扩容，而在请求高峰时段过去后， Pod CPU 利用率又会降下来，此时对应的 Pod 副本数应该自动减少到一个合理的水平。如果目标 Pod 没有定义 Pod Request 值，则无法使用 CPUUtilizationPercentage 实现 Pod 横向自动扩容。除了使用 CPUUtilizationPercentage，Kubernetes 从 1.2 版本开始也在尝试支持应用程序自定义的度量指标。

在 CPUUtilizationPercentage 计算过程中使用到的 Pod 的 CPU 使用量通常是 1 min 内的平均值，通常通过查询 Heapster 监控子系统来得到这个值，所以需要安装部署 Heapster，这样便增加了系统的复杂度和实施 HPA 特性的复杂度。因此，从 1.7 版本开始，Kubernetes 自身孵化了一个基础性能数据采集监控框架 --- Kubernetes Monitoring Architecture，从而更好地支持 HPA 和其他需要用到基础性能数据的功能模块。在 Kubernetes Monitoring Architecture 中，Kubernetes 定义了一套标准化的 API 接口 Resource Metrics API，以方便客户端应用程序（如 HPA）从 Metrics Server 中获取目标资源对象的性能数据，例如容器的 CPU 和内存使用数据。到 Kubernetes 1.8 版本，Resource Metrics API 被升级为 metrics.k8s.io/v1beta1，已经接近生产环境中的可用目标了。


## 示例

下面是 HPA 定义的一个具体例子：
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    kind: Deployment
    name: php-apache
  targetCPUUtilizationPercentage: 90
```

根据上面的定义，我们可以知道这个 HPA 控制的目标对象为一个名为 php-apache 的 Deployment 里的 Pod 副本，当这些 Pod 副本的 CPUUtilizationPercentage 的值超过 90% 时会触发自动动态扩容行为，在扩容或缩容时必须满足一个约束条件是 Pod 副本数量为 1 ~ 10。

除了可以通过直接定义 YAML 文件并且调用 kubectl create 命令来创建一个 HPA 资源对象的方式，还可以通过下面的简单命令行直接创建等价的 HPA 对象：
```bash
kubectl autoscale deployment php0apache --cpu-percent=90 --min=1 --max=10
```

第 2 章将会给出一个完整的 HPA 例子来说明其用法和功能。
