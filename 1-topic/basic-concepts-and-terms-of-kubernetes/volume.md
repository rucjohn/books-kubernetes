# Volume

Volume（存储卷）是 Pod 中能够被多个容器访问的共享目录。Kubernetes 的 Volume 概念、用途和目的与 Docker 的 Volume 比较类似，但两者不能等价。
* 首先，Kubernetes 中的 Volume 被定义在 Pod 上，然后被一个 Pod 里的多个容器挂载到具体的文件目录下；
* 其次，Kubernetes 中的 Volume 与 Pod 的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，Volume 中的数据不会丢失。
* 最后，Kubernetes 支持多种类型的 Volume，例如 GlusterFS、Ceph 等先进的分布式文件系统。

Volume 的使用比较简单，在大多数情况下，我们先在 Pod 上声明一个 Volume，然后在容器里引用该 Volume 并挂载到容器的某个目录上。举例来说，我们要给之前的 Tomcat Pod 增加一个名为 datavol 的 Volume，并且挂载到容器的 /mydata-data 目录上，则只要对 Pod 的定义文件如下修改即可。
```yaml
......
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: tomcat-demo
        image: tomcat
        volumeMounts:
        - mountPath: /mydata-data
          name: datavol
        imagePullPolicy: IfNotPresent
```

除了可以让一个 Pod 里多个容器共享文件、让容器的数据写到宿主机的磁盘上或者写文件到网络存储中，Kubernetes 的 Volume 还扩展出一种非常实用价值的功能，即容器配置文件集中化定义与管理，这是通过 ConfigMap 这种新的资源对象来实现的，后面会详细说明。

Kubernetes 提供了非常丰富的 Volume 类型，下面逐一进行说明。

## emptyDir

一个 emptyDir Volume 是在 Pod 分配到 Node 时创建的。从它的名称就可以看出，它的初始化内容空，并且无须指定宿主机上对应的目录文件，因为这是 Kubernetes 自动分配的一个目录，当 Pod 从 Node 上移除时，emptyDir 中的数据也会被永久删除。emptyDir 的一些用途如下：
* 临时空间，例如用于某些应用程序运行所需的临时目录，且无须永久保留。
* 长时间任务的中间过程 CheckPoint 的临时保存目录。
* 一个容器需要另一个容器中获取数据的目录（多容器共享目录）。

目前，用户无法控制 emptyDir 使用介质类型。如果 kubelet 的配置使用硬盘，那么所有 emptyDir 都将被创建在该硬盘上。Pod 在将来可以设置 emptyDir 是位于硬盘、固态硬盘还是基于内存的 tmpfs 上，上面的例子便采用了 emptyDir 类的 Volume。

## hostPath

hostPath 为在 Pod 上挂载宿主机上的文件或目录，它通常可用于以下几个方面：
* 容器应用程序生成的日志文件需要记录保存时，可以使用宿主机的高速文件系统进行存储。
* 需要访问宿主机上 Docker 引擎内部数据结构的容器应用时，可以通过定义 hostPath 为宿主机 /var/lib/docker 目录，使容器内部应用可以直接访问 Docker 的文件系统。

在使用这种类型的 Volume 时，需要注意以下几点：
* 在不同的 Node 上具有相同配置的 Pod，可能会因为宿主机上的目录和文件不同而导致对 Volume 上目录和文件的访问结果不一致。
* 如果使用了资源配额管理，则 Kubernetes 无法将 hostPath 在宿主机上使用的资源纳入管理。

下面的例子中使用宿主机的 /data 目录定义了一个 hostPath 类型的 Volume：
```yaml
volumes:
- name: "persistent-storage"
  hostPath: 
    path: "/data"
```

## gcePersistentDisk

使用这种类型的 Volume 表示使用谷歌公有云提供的永久磁盘（Persistent Disk，PD）存放 Volume 数据，它与 emptyDir 不同，PD 上的内容会被永久保存，当 Pod 被删除时，PD 只是被卸载，但不会被删除。需要注意的是，你需要先创建一个 PD，才能使用 gcePersistentDisk。

使用 gcePersistentDisk 时有以下一些限制：
* Node（运行 kubelet 的节点）需要是 GCE 虚拟机。
* 这些虚拟机需要与 PD 存在相同的 GCE 项目和 Zone 中。

通过 gcloud 命令即可创建一个 PD：
```bash
gcloud compute disks create --size=500GB --zone=us-centrall-a my-data-disk
```

定义 gcePersistentDisk 类型的 Volume 的示例如下：
```yaml
volumes:
- name: gce-volume
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
```

## awsElasticBlockStore

与 GCE 类似，该类型的 Volume 使用 AWS EBS Volume 存储数据，需要先创建一个 EBS Volume 才能使用 awsElasticBlockStore。

使用 awsElasticBlockStore 的一些限制条件如下：
* Node（运行 kubelet 的节点）需要是 AWS EC2 实例。
* 这些 AWS EC2 实例需要与 EBS Volume 存在于相同的 region 和 availability-zone 中。
* EBS 只支持单个 EC2 实例挂载一个 Volume。

通过以下命令可以创建一个 EBS Volume：
```bash
aws ec2 create-volume --availability-zone eu-west-1a --size 10 --volume-type gp2
```

定义 awsElasticBlockStore 类型的 Volume 的示例如下：
```yaml
volumes:
- name: aws-volume
  awsElasticBlockStore:
    volumeID: aws://<availability-zone>/<volume-id>
    fsType: ext4
```

## NFS

使用 NFS 网络文件系统提供的共享目录存储数据时，我们需要在系统中部署一下 NFS Server。定义 NFS 类型的 Volume 的示例如下：
```yaml
volumes:
- name: nfs-volume
  nfs:
    server: <nfs-server>
    path: "/"
```

## 其他类型的 Volume

* iscsi：使用 iSCSI 存储设备上的目录挂载到 Pod 中。
* flocker：使用 Flocker 管理存储卷。
* glusterfs：使用开源 GlusterFs 网络文件系统的目录挂载到 Pod 中。
* rbd：使用 Ceph 块设备共享存储（Rados Block Device）挂载中 Pod 中。
* gitRepo：通过挂载一个空目录，并从 Git 库 clone 一个 git repository 以供 Pod 使用。
* secret：一个 Secert Volume 用于为 Pod 提供加密信息，你可以将定义在 Kubernetes 中的 Secert 直接挂载为文件让 Pod 访问。Secert Volume 是通过 TMFS（内存文件系统）实现的，这种类型的 Volume 总是不会被持久化的。

