---
template: main.html
tags:
  - K8S
  - Kubernetes
  - Resource
---

# Deployment

- 为 Pod 和 ReplicaSet 提供声明式的更新能力

## 创建

```bash
zsh ➜ kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

## 查询

```bash
# 检查 Deployment 是否已创建
zsh ➜ kubectl get deployments

# 查看 Deployment 上线状态
zsh ➜ kubectl rollout status deployment/nginx-deployment

# 查看 Deployment 创建的 ReplicaSet
zsh ➜ kubectl get rs

# 查看 Deployment 创建的 pod
zsh ➜ kubectl get pods --show-labels
```

## 更新

```bash
# 更新镜像
zsh ➜ kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
# OR: kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
# OR: kubectl edit deployment/nginx-deployment

# 查看 Deployment 更新过程
zsh ➜ kubectl describe deployments
```

## 更新策略

- .spec.strategy.type
  - RollingUpdate （default）
    - 指定 maxUnavailable 和 maxSurge 来控制滚动更新 过程
    - .spec.strategy.rollingUpdate.maxUnavailable 更新过程中不可用的 Pod 的个数上限
    - .spec.strategy.rollingUpdate.maxSurge 可以创建的超出期望 Pod 个数的 Pod 数量
  - Recreate
    - 在创建新 Pods 之前，所有现有的 Pods 会被杀死

## 回滚

```bash
# 更新操作&版本记录
zsh ➜ kubectl set image deployment/nginx-deployment nginx=nginx:1.161 --record=true

# 检查操作版本
zsh ➜ kubectl rollout history deployment/nginx-deployment

# 检查版本详细信息
zsh ➜ kubectl rollout history deployment/nginx-deployment --revision=2

# 回滚到之前的修订版本
zsh ➜ kubectl rollout undo deployment/nginx-deployment
zsh ➜ OR: kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## 缩/扩容

```bash
# 指定replica数量
zsh ➜ kubectl scale deployment/nginx-deployment --replicas=5

# 自动缩/扩容
zsh ➜ kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80

# 比例缩放
# RollingUpdate 的 Deployment 支持同时运行应用程序的多个版本。 当自动缩放器缩放处于上线进程（仍在进行中或暂停）中的 RollingUpdate Deployment 时，
# Deployment 控制器会平衡现有的活跃状态的 ReplicaSets（含 Pods 的 ReplicaSets）中的额外副本， 以降低风险
zsh ➜ kubectl set image deployment/nginx-deployment nginx=nginx:sometag
zsh ➜ kubectl get rs
```

## 暂停、恢复 Deployment 的上线过程

```bash
# 暂停 Deployment 的上线过程，能够在暂停和恢复执行之间应用多个修补程序，而不会触发不必要的上线操作
zsh ➜ kubectl rollout pause deployment/nginx-deployment
zsh ➜ kubectl rollout resume deployment/nginx-deployment
```

# ReplicaSet

- 维护一组 Pod 副本的稳定集合

# StatefulSet

- 管理有状态应用的工作负载 API 对象、管理某 Pod 集合的部署和扩缩， 并为这些 Pod 提供持久存储和持久标识符

## 常见使用场景

- 稳定的、唯一的网络标识符
- 稳定的、持久的存储
- 有序的、优雅的部署和扩缩
- 有序的、自动的滚动更新

# DaemonSet

- 确保全部（或者某些）节点上运行一个 Pod 的副本

## 常见使用场景

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

## 创建

```bash
zsh ➜ kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

# Job

- Job 会创建一个或者多个 Pod，并将继续重试 Pod 的执行，直到指定数量的 Pod 成功终止。 随着 Pod 成功结束，Job 跟踪记录成功完成的 Pod 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束
- 删除 Job 的操作会清除所创建的全部 Pod

## 创建 job

```bash
zsh ➜ kubectl apply -f https://kubernetes.io/examples/controllers/job.yaml
```

## Job 的并行执行

- 非并行 Job
  - 通常只启动一个 Pod，除非该 Pod 失败, 当 Pod 成功终止时，立即视 Job 为完成状态
- 具有确定完成计数的并行 Job
  - .spec.completions 字段设置为非 0 的正数值
  - NoIndexed
    - 当成功的 Pod 个数达到 .spec.completions 时，Job 被视为完成
  - Indexed
    - 当每个索引都对应一个完成完成的 Pod 时，Job 被认为是已完成的。
- 带工作队列的并行 Job
  - .spec.parallelism(在任意时刻运行状态的 Pod 个数) 控制并行性
  - 多个 Pod 之间必须相互协调，或者借助外部服务确定每个 Pod 要处理哪个工作条目
  - **每个 Pod 都可以独立确定是否其它 Pod 都已完成，进而确定 Job 是否完成**
  - 当 Job 中任何 Pod 成功终止，不再创建新 Pod，剩下的 Pod 允许执行完毕
  - 一旦至少 1 个 Pod 成功完成，并且所有 Pod 都已终止，即可宣告 Job 成功完成
  - 一旦任何 Pod 成功退出，任何其它 Pod 都不应再对此任务执行任何操作或生成任何输出。所有 Pod 都应启动退出过程

## 容错

- 通过 .spec.template.spec.restartPolicy 设置重启策略
- 应用需要处理在一个新 Pod 中被重启的情况
- 即使将 .spec.parallelism 设置为 1，且将 .spec.completions 设置为 1，并且 .spec.template.spec.restartPolicy 设置为 "Never"，同一程序仍然有可能被启动两次

## Job 终止与清理

- Job 完成时不会再创建新的 Pod，不过已有的 Pod 通常也不会被删除；可以使用 kubectl 来删除 Job
- 终止 Job
  - 为 Job 的 .spec.activeDeadlineSeconds 设置一个秒数值。一旦 Job 运行时间达到 activeDeadlineSeconds 秒，其所有运行中的 Pod 都会被终止。
- 自动清理 Job
  - 设置 Job 的 .spec.ttlSecondsAfterFinished 字段，可以让该控制器清理掉已结束的资源。

## 挂起 Job

- 更新 .spec.suspend 字段为 true 挂起 Job，恢复其执行时，将其更新为 false；挂起 Job 会删除其所有活跃的 Pod

# CronJob

- 创建基于时隔重复调度的 Jobs

## 限制

- Job 应该是幂等的

# ReplicationController

- 确保在任何时候都有特定数量的 Pod 副本处于运行状态。换句话说，ReplicationController 确保一个 Pod 或一组同类的 Pod 总是可用的。

## 创建

```bash
zsh ➜ kubectl apply -f https://k8s.io/examples/controllers/replication.yaml
```
