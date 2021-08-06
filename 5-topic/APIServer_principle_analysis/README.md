# Kubernetes API Server 原理解析

总体来看，Kubernetes API Server 的核心功能是提供 Kubernetes 各类资源对象（如 Pod、RC、Service等）的增、删、改、查及 Watch 等HTTP REST 接口，成为集群内各个功能模块之间数据交互和通信的中心枢纽，

