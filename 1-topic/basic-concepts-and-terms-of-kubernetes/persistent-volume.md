# Persistent Volume

之前提供的 Volume 是被定义在 Pod 上的，属于计算资源的一部分，而实际上，网络存储是相对独立于计算资源而存在的一种实体资源。比如在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中划出一个『网盘』并挂接到虚拟机上。Persistent Volume（PV）和与之相关联的 Persistent Volume Claim（PVC）也起到了类似的作用。

PV 可以被理解成 Kubernetes 集群中的某个网络存储对应的一块存储，它与 Volume 类似，但有以下区别：
* PV 只能是网络存储，不属于任何 Node，但可以在每个 Node 访问。
* PV 并不是被定义在 Pod 上的，而是独立于 Pod 之外定义的。
* PV 目前支持类型包括：
  * gcePersistentDisk
  * awsElasticBlockStore
  * AzureFile
  * AzureDisk
  * FC（Fibre Channel）
  * Flocker
  * NFS
  * iSCSI
  * RBD（Rados Block Device）
  * CephFS
  * Cinder
  * GlusterFs
  * VsphereVolume
  * Quobyte Volumes
  * VMware Photon
  * Portworx Volumes
  * ScaleIO Volumes
  * HostPath

下面给出了 NFS 类型的 PV 的一个 YAML 定义文件，声明了需要 5Gi 的存储空间：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /sompath
    server: 172.17.0.2
```

比如重要的 PV 的 accessModes 属性，目前以下类型：
* ReadWriteOnce：读写权限，并且只能被单个 Node 挂载
* ReadOnlyMany：只读权限，允许被多个 Node 挂载
* ReadWriteMany：读写权限，允许被多个 Node 挂载

如果 Pod 想申请某种类型的 PV，则首先需要定义一个 PersistentVolumeChaim 对象：
```yaml
apiVersion: v1
kind: PersistentVolumeChaim
metadata:
  name: mychaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

然后，在 Pod 的 Volume 定义中引用上述 PVC 即可：
```yaml
volumes:
- name: mypd
  persistentVolumeChaim:
    chaimName: myclaim
```

最后说说 PV 的状态。PV 是有状态的对象，它的状态有以下几种：
* Available：空闲状态
* Bound：已经绑定到某个 PVC 上。
* Released：对应的 PVC 已经被删除，但资源还没有被集群收回。
* Failed：PV 自动回收失败。

共享存储的原理解析和实践指南详见第 8 章的说明。
