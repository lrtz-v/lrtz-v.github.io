---
template: main.html
tags:
  - K8S
  - Kubernetes
---

# Kubernetes 安装（On Apple M1）

- doc: https://kubernetes.io/zh-cn/docs/home/

## minikube 安装

```bash
zsh ➜ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64

zsh ➜ sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

## kubectl 安装

```bash
zsh ➜ brew install kubectl
```

## Kubernetes 集群启动

```bash
# 集群启动
zsh ➜ minikube start
# 集群停止
zsh ➜ minikube stop

# Kubernetes dashboard
zsh ➜ minikube dashboard

# Upgrade your cluster
zsh ➜ minikube start --kubernetes-version=latest

# 集群暂停
zsh ➜ minikube pause
# 集群恢复
zsh ➜ minikube unpause

# delete cluster
zsh ➜ minikube delete

# Delete all local clusters and profiles
zsh ➜ minikube delete --all
```
