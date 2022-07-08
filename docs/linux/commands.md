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

- 端口占用

```bash
lsof -i:3306
```

- 查看进程打开的文件

```bash
lsof -p 56863
```

- 查看指定目录下被进程开启的文件

```bash
lsof +d /tmp
```

- 查看用户 username 的进程所打开的文件

```bash
lsof -u username
```

- 查询归属于用户 username 的进程

```bash
ps -lu username
```

## top 命令

```bash
# top
Processes: 469 total, 2 running, 467 sleeping, 3111 threads                                                                                                                     15:32:48
Load Avg: 2.57, 2.46, 2.81  CPU usage: 9.67% user, 10.82% sys, 79.49% idle   SharedLibs: 438M resident, 89M data, 23M linkedit.
MemRegions: 348979 total, 2041M resident, 149M private, 2422M shared. PhysMem: 15G used (1960M wired), 83M unused.
VM: 188T vsize, 3823M framework vsize, 8716015(0) swapins, 10359236(0) swapouts. Networks: packets: 63434144/53G in, 46790060/28G out. Disks: 22489480/544G read, 13429524/440G written.

PID    COMMAND      %CPU TIME     #TH   #WQ  #PORT MEM    PURG   CMPRS  PGRP  PPID  STATE    BOOSTS             %CPU_ME %CPU_OTHRS UID  FAULTS     COW     MSGSENT     MSGRECV
38123  com.apple.Vi 25.6 90:48.24 11    1    61    8084M  0B     9273M- 38123 1     sleeping *4[3]              0.00000 0.00000    501  6363101+   92      490         186
388    WindowServer 24.2 15:34:57 23    6    3884  981M-  5920K+ 274M   388   1     sleeping *0[1]              4.67062 2.97198    88   55674126+  9413740 404174596+  288000142+
14642  Notion Helpe 20.2 02:03:43 17    1    207   433M+  0B     373M-  14632 14632 sleeping *0[121]            0.00000 0.00000    501  9917166+   225816  9774063+    4216281+
0      kernel_task  19.3 09:23:30 508/8 0    0     165M+  0B     0B     0     0     running   0[0]              0.00000 0.00000    0    53925      0       1398020956+ 1170512637+
29385  iTerm2       14.9 16:01.83 10    7    415-  378M-  76M-   133M-  29385 1     sleeping *0[4915]           0.74361 1.26299    501  9827951+   421     1833342+    386319+
```

- 常见搭配

```bash
P：根据CPU使用百分比大小进行排序
M：根据驻留内存大小进行排序
i：使top不显示任何闲置或者僵死进程
```
