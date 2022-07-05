---
tags:
  - Linux
  - Centos
  - Kernel
---

# Centos 内核升级

## 系统更新

```bash
[root@localhost ~]$ yum -y update
[root@localhost ~]$ yum -y install yum-plugin-fastestmirror
```

## 检查当前系统及内核版本

```bash
[root@localhost ~]$ cat /etc/redhat-release
CentOS release 7.3 (Final)

[root@localhost ~]$ uname -r
3.10.0_3-0-0-30
```

## 新增 repo：elrepo

```bash
[root@localhost ~]$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@localhost ~]$ rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum repolist
```

## 安装新版本内核

```bash
[root@localhost ~]$ yum --enablerepo=elrepo-kernel install kernel-ml
[root@localhost ~]$ yum repolist all
```

## 配置启动项

### 查询可用内核启动项

```bash
[root@localhost ~]$ sudo awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

### 设置启动项并生成启动项配置

```bash
sudo grub2-set-default 0
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

## 重启系统

```bash
[root@localhost ~]$ sudo reboot
```

## 后续

### 检查内核版本

```bash
[root@localhost ~]$ uname -r
```

### 移除旧版本内核

```bash
[root@localhost ~]$ yum install yum-utils
[root@localhost ~]$ package-cleanup --oldkernels
```
