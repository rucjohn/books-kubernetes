# 启动 Tomcat 服务

上面定义和启动了 MySQL 服务，接下来采用同样的步骤完成 Tomcat 应用的启动过程。首先，创建对应的 RC 文件 `myweb-rc.yaml`，内容如下：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 1
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
```

注意：在 Tomcat 容器内，应用将使用环境变量 MYSQL\_SERICE\_HOST 的值连接MySQL 服务。更安全可靠的用户是使用服务的名称 mysql，详见本章 Service 的概念和第4章的说明。运行下面的命令，完成 RC 的创建和验证工作：

```bash
kubectl create -f myweb-rc.yaml
kubectl get pods
```

结果输出

```text
NAME          READY   STATUS    RESTARTS   AGE
mysql-vgzw4   1/1     Running   0          24h
myweb-l5m6z   1/1     Running   0          2m28s
```

最后，创建对应的 Service。以下是完整的YAML定义文件（myweb-svc.yaml）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
  selector:
    app: myweb
```

`type=NodePort` 和 `nodePort=3001` 的两个属性表时此 Service 开启了 NodePort 方式的外网访问模式。在 Kubernetes 集群之外，比如在本机的浏览器里，可以通过 30001 这个端口访问 myweb（对应到8080的虚端口上）。运行 kubectl create 命令进行创建：

```bash
kubectl create -f myweb-svc.yaml 
kubectl get svc
```

查看创建的 Service：

```text
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          47h
mysql        ClusterIP   10.111.135.20   <none>        3306/TCP         23h
myweb        NodePort    10.97.252.206   <none>        8080:30001/TCP   6s
```

到此，我们的第1个 Kubernetes 例子便搭建完成了，我们将在下一节中难结果。

