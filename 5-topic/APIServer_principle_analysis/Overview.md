
# 概述

Kubernetes API Server 通过一个名为 kube-apiserver 的进程提供服务，该进程运行在控制平面上。

在默认情况下，kube-apiserver 进程：
- 绑定 8080 端口（`--insecure-port`）提供 HTTP 服务。
- 绑定 6443 端口（`--secure-port`）提供 HTTPS 服务，加强 REST API 访问的安全性。

## kubelet 方式

在控制平面上通常可以通过命令行工具 `kubectl` 来与 API Server 交互，它们之间的接口是 RESTful API。

## curl 方式

为了测试和学习 Kubernetes API Server 所提供的接口，也可以使用 `curl` 命令进行快速验证。比如，登录控制平面并运行以下命令，将得到以 JSON 方式返回的 Kubernetes API 版本信息：
```
curl localhost:8080/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.18.131:6443"
    }
  ]
}
```

可以运行下面的命令查看 Kubernetes API Server 目录支持的资源对象的种类：
```
curl localhost:8080/api/v1
```

根据以上命令的输出，可以运行下面的命令，分别返回集群中的 Pod列表、Service 列表、RC 列表等：
```
curl localhost:8080/api/v1/pods
curl localhost:8080/api/v1/services
curl localhost:8080/api/v1/replicationcontrollers
```

如果只想对外暴露部分 REST 服务，则可以在控制平面或其他节点上运行 `kubectl proxy` 进程启动一个内部代理来实现。运行以下命令，在 8001 端口启动代理，并且拒绝客户端访问 RC 的 API：
```
kubectl proxy --reject-paths="^/api/v1/replicationcontrollers" --port=8001 --v=2
```

验证
```
curl localhost:8080/api/v1/replicationcontrollers

<h3>Unauthorized</h3>
```

kubectl proxy 具有很多特性，最实用的一个特性是提供简单有效的安全机制，比如在采用白名单限制非法客户端访问时，只需要增加下面这个参数即可：
```
--accept-hosts="^localhost$,^127\\.0\\.0\\.1$,^\\[::1\\]$"
```

