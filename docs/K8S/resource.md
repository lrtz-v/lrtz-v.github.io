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
