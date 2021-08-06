# Scheduler 原理解析

Kubernetes Scheduler 在整个系统中承担了 “承上启下” 的重要功能：
- “承上” 是指它负责接收 Controller Manager 创建的新 Pod，为其安排一个落脚的 “家” --- 目标 Node。
- “启下” 是指安置工作完成后，目标 Node 上的 kubelet 服务进程接管后继工作，负责 Pod 生命周期中的 “下半生”。
