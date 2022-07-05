---
template: main.html
tags:
  - Linux
---
# max file descriptors

## /etc/security/limits.conf

- 添加如下配置

```shell
* soft nofile 655360
* hard nofile 655360
* soft nproc 655360
* hard nproc 655360
```

## /etc/sysctl.conf (root)

- 添加如下配置

```shell
vm.max_map_count=655360
```

- 执行

```bash
sysctl -p
```

## 磁盘挂载

```shell
查看磁盘使用情况：df -h

查看磁盘挂载：fdisk -l
格式化磁盘：mkfs.xfs -f /dev/vdb
创建挂载路径：mkdir /data
挂载：mount -t xfs /dev/vdb /data

固定配置：
vim /etc/fstab
/dev/vdb /data xfs defaults 0 0
```

## 常用命令

- 查看物理 CPU

```bash
cat /proc/cpuinfo | grep "physical"| sort |uniq -c
```

- 查看服务器型号

```bash
dmidecode | grep “Product Name”
```

- 查看系统版本

```bash
cat /etc/redhat-release
```

- 查看分区和磁盘

```bash
lsblk
```

- 查看磁盘空间使用情况

```bash
df -h
```

- 分区工具查看分区信息

```bash
fdisk -l
```

- 查看分区

```bash
cfdisk /dev/sda
```

- 查看硬盘 label（别名）

```bash
blkid
```

- 统计当前目录各文件夹大小

```bash
du -sh ./*
```

- 查看内存大小

```bash
free -h
```

- 查看当前网络连接情况

```bash
netstat -ant|awk '/^tcp/ {++S[$NF]} END {for(a in S) print (a,S[a])}'
```

- 查看 CPU 核数和型号和主频

```bash
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
```

- 查看 cpu 核心数

```bash
cat /proc/cpuinfo| grep "cpu cores"| uniq
```

- 查看物理 cpu 个数

```bash
cat /proc/cpuinfo| grep "physical id"|uniq| wc -l
```

- 查看逻辑 cpu 的个数

```bash
cat /proc/cpuinfo| grep "processor"| wc -l
```
