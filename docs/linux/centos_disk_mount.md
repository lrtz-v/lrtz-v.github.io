---
tags:
  - Linux
  - Centos
  - Dick
template: main.html
---

# Centos7 磁盘挂载

## 查看磁盘挂载

```bash
# fdisk -l

Disk /dev/sda: 4000.8 GB, 4000787030016 bytes, 7814037168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: gpt
Disk identifier: ...

....
```

## 磁盘分区

```bash
# fdisk /vdb/sda

Command action
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help): n (新建分区)
```

## 磁盘格式化

```bash
mkfs.ext4 /vdb/sda
```

## 磁盘挂载

```bash
mount /vdb/sda  /<目的目录>
```

## 持久化挂载 /etc/fstab

```vim
/vdb/sda  /<目的目录> ext4 defaults 0 0
```
