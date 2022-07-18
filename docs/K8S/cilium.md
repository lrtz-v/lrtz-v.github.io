---
template: main.html
tags:
  - K8S
  - Kubernetes
  - Cilium
---

# Cilium 安装

- doc: https://docs.cilium.io/en/stable/gettingstarted/http/

## 启动 Kubernetes 集群，设置 cni 为 cilium

```bash
zsh ➜ minikube start --network-plugin=cni --cni=cilium
```

## Cilium CLI 安装

```bash
zsh ➜ wget https://github.com/cilium/cilium-cli/releases/download/v0.11.11/cilium-darwin-arm64.tar.gz
sudo mv cilium /usr/local/bin
```

## 安装 Cilium

```bash
zsh ➜ cilium install
🔮 Auto-detected Kubernetes kind: minikube
✨ Running "minikube" validation checks
✅ Detected minikube version "1.26.0"
ℹ️  Using Cilium version 1.11.6
🔮 Auto-detected cluster name: minikube
🔮 Auto-detected datapath mode: tunnel
ℹ️  helm template --namespace kube-system cilium cilium/cilium --version 1.11.6 --set cluster.id=0,cluster.name=minikube,encryption.nodeEncryption=false,kubeProxyReplacement=disabled,operator.replicas=1,serviceAccounts.cilium.name=cilium,serviceAccounts.operator.name=cilium-operator,tunnel=vxlan
ℹ️  Storing helm values file in kube-system/cilium-cli-helm-values Secret
🔑 Created CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.11.6...
🚀 Creating Agent DaemonSet...
level=warning msg="spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[1].matchExpressions[0].key: beta.kubernetes.io/os is deprecated since v1.14; use \"kubernetes.io/os\" instead" subsys=klog
🚀 Creating Operator Deployment...
⌛ Waiting for Cilium to be installed and ready...
✅ Cilium was successfully installed! Run 'cilium status' to view installation health


zsh ➜ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet         cilium             Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium             Running: 1
                  cilium-operator    Running: 1
Cluster Pods:     1/1 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.11.6@sha256:f7f93c26739b6641a3fa3d76b1e1605b15989f25d06625260099e01c8243f54c: 1
                  cilium-operator    quay.io/cilium/operator-generic:v1.11.6@sha256:9f6063c7bcaede801a39315ec7c166309f6a6783e98665f6693939cf1701bc17: 1
```

## hubble 安装

```bash
zsh ➜ cilium hubble enable --ui

zsh ➜ wget https://github.com/cilium/hubble/releases/download/v0.10.0/hubble-darwin-arm64.tar.gz
zsh ➜ tar xvf hubble-darwin-arm64.tar.gz
zsh ➜ sudo mv hubble /usr/local/bin
zsh ➜ cilium hubble port-forward&
zsh ➜ hubble status
zsh ➜ hubble observe

# open web UI
zsh ➜ cilium hubble ui
```
