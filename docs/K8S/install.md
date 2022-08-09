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

## prod deployment（centos 7 & kubeadm）

### 系统初始化

- 停止防火墙，Iptables 设置为空

```bash
zsh ➜ systemctl  stop firewalld  &&  systemctl  disable firewalld
zsh ➜ yum -y install iptables-services  &&  systemctl  start iptables  &&  systemctl  enable iptables &&  iptables -F  &&  service iptables save
```

- 关闭不必要的服务

```bash
zsh ➜ swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
zsh ➜ setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
zsh ➜ systemctl stop postfix && systemctl disable postfix
```

- 关闭 NUMA

```bash
zsh ➜ cp /etc/default/grub{,.bak}
zsh ➜ vim /etc/default/grub # 在 GRUB_CMDLINE_LINUX 一行添加 `numa=off` 参数，如下所示: diff /etc/default/grub.bak /etc/default/grub
6c6
< GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet"
---
> GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet numa=off" cp /boot/grub2/grub.cfg{,.bak}
grub2-mkconfig -o /boot/grub2/grub.cfg
```

- /etc/sysctl.d/kubernetes.conf

```conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
```

```bash
zsh ➜ sysctl -p /etc/sysctl.d/kubernetes.conf
```

- 时区设置

```bash
zsh ➜ timedatectl set-timezone Asia/Shanghai
zsh ➜ timedatectl set-local-rtc 0
zsh ➜ systemctl restart rsyslog
zsh ➜ systemctl restart crond
```

- 设置 rsyslogd 和 systemd journald

```bash
zsh ➜  mkdir /var/log/journal # 持久化保存日志的目录
zsh ➜  mkdir /etc/systemd/journald.conf.d
zsh ➜ cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 压缩历史日志
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 不将日志转发到
syslog ForwardToSyslog=no
EOF

zsh ➜ systemctl restart systemd-journald
```

- 内核升级 4.44+
  - https://lrtz-v.github.io/linux/centos_kernal_upgrade/

### 使用 Kubeadm 部署

- ipvs 设置

```bash
zsh ➜ modprobe br_netfilter
zsh ➜ cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
zsh ➜ chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

- docker

```bash
zsh ➜ yum install -y yum-utils device-mapper-persistent-data lvm2
zsh ➜ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
zsh ➜ yum update -y && yum install -y docker-ce ## 创建 /etc/docker 目录
zsh ➜ mkdir /etc/docker
zsh ➜ cat > /etc/docker/daemon.json <<EOF {
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  }
}
EOF
zsh ➜ mkdir -p /etc/systemd/system/docker.service.d
zsh ➜ systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

- Kubeadm

```bash
zsh ➜ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
zsh ➜ yum -y  install  kubeadm-1.22 kubectl-1.22 kubelet-1.22
zsh ➜ systemctl enable kubelet.service
```

- master 节点初始化

```bash
zsh ➜ kubeadm config print init-defaults > kubeadm-config.yaml
    localAPIEndpoint:
        advertiseAddress: 192.168.66.10
    kubernetesVersion: v1.15.1
    networking:
      podSubnet: "10.244.0.0/16"
      serviceSubnet: 10.96.0.0/12
    ---
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    featureGates:
      SupportIPVSProxyMode: true
    mode: ipvs
zsh ➜ kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs | tee kubeadm-init.log
```

- 节点加入

```bash
zsh ➜ kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

- 网络通信

```bash
zsh ➜ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
