# Job

批处理任务通常并行（或者串行）启动多个计算进程去处理一批工作项（work item），在处理完成后，整个批处理任务结束。从 1.2 版本开始，Kubernetes 支持批处理类型的应用，我们可以通过 Kubernetes Job 这种新的资源对象定义并启动一个批处理任务 Job。与RC、Deployment、ReplicaSet、DaemonSet类似，Job 也控制一组 Pod 容器。从这个角度来看，Job 也是一种特殊的 Pod 副本自动控制器，同时 Job 控制 Pod 副本与 RC 等控制器的工作机制有以下重要差别：
* Job 所控制的 Pod 副本是短暂运行的，可以将其视为一组 Docker 容器，其中的每个 Docker 容器都仅仅运行一次。当 Job 控制的所有 Pod 副本都运行结束时，对应的 Job 也就结束了。Job 在实现方式上与 RC 等副本控制器不同，Job 生成的 Pod 副本是不能自动重启的，对应的 Pod 副本的 RestartPolicy 都被设置为 Never。因此，Job 生成的 Pod 副本都执行完成时，相应的 Job 也就完成了控制使命，即 Job 生成的 Pod 在 Kubernetes 中是短暂存在的。Kubernetes 在 1.5 版本之后又提供类似 crontab 的定时任务 --- CronJob，解决了某些批处理任务需要定时反复执行的问题。
* Job 所控制的 Pod 副本的工作模式能够多实例并行计算，以 TensorFlow 框架为例，可以将一个机器学习的计算任务分布到 10 台机器上，在每台机器上都运行一个 worker 执行计算任务，这很适合通过 Job 生成 10 个 Pod 副本同时启动运算。

在第 3 章会继续深入讲解 Job 的实现原理及对应的案例。

