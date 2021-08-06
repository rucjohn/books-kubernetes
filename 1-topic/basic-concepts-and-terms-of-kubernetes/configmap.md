# ConfigMap

为了能够准确和深刻理解 Kubernetes ConfigMap 的功能和价值，我们需要从 Docker 说起。我们知道，Docker 通过将程序、依赖库、数据及配置文件『打包固化』到一个不变的镜像文件中的做法，解决了应用的部署的难题，但这同时带来了棘手的问题，即配置文件中的参数在运行期间如何修改的问题。我们不可能在启动 Docker 容器后再修改容器里的配置文件，然后用新的配置文件重启容器里的用户主进程。为了解决这个问题，Docker 提供了两种方式：
* 在运行时通过容器的环境变量来传递参数；
* 通过 Docker Volume 将容器外的配置文件映射到容器内；

这两种方式都其优势和缺点，在大多数情况下，后一种方式更合适我们的系统，因为大多数应用通常从一个或多个配置文件中读取参数。但这种方式也有明显的缺陷：我们必须在目标主机上先创建好对应的配置文件，然后才能映射到容器中。

上述缺陷在分布式情况下变得更严重，因为无论采用哪种方式，写入（修改）多台服务器上的某个指定文件，并确保这些文件保持一致，都是一个很难完成的目标。此外，在大多数情况下，我们都希望能集中管理系统的配置参数，而不是管理一堆配置文件。针对上述问题，Kubernetes 给出一个巧妙的设计实现，如下：

首先，把所有的配置项都当作 k/v 字符串，当然 value 可以来自某个文本文件，比如配置项 password=123456、user=root、host=192.168.2.1 用于表示连接 ftp 服务器的配置参数。这些配置项可以作为 Map 表中的一个项，整个 Map 的数据可以被持久化存储在 Kubernetes 的 etcd 数据库中，然后提供 API 以方便 Kubernetes 相关组件或客户应用 CRUD 操作这些数据，上述专门用来保存配置参数的 Map 就是 Kubernetes ConfigMap 资源对象。

接下来，Kubernetes 提供了一种内建机制，将存储在 etcd 中的 ConfigMap 通过 Volume 映射的方式变成目标 Pod 内的配置文件，不管目标 Pod 被调度到哪台服务器上，都会完成自动映射。进一步地，如果 ConfigMap 中的 k/v 数据被修改，则映射到 Pod 中的 『配置文件』也会随之自动更新。于是，Kubernetes ConfigMap 就成了分布式系统中最为简单（使用方法简单，但背后实现比较复杂）且对应用无侵入的配置中心。

