# Kubernetes API Server 原理解析

总体来看，Kubernetes API Server 的核心功能是提供 Kubernetes 各类资源对象（如 Pod、RC、Service等）的增、删、改、查及 Watch 等HTTP REST 接口，成为集群内各个功能模块之间数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心。除此之外，还有以下一些功能特性：
- 是集群管理的 API 入口
- 是资源配额控制的入口
- 提供了完备的集群安全机制

