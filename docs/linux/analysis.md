---
template: main.html
tags:
  - Linux
---

# 性能分析

## 平均负载

- 执行 top 或者 uptime 命令，可以看到如下信息

  ```bash
    21:40  up 44 days, 49 mins, 3 users, load averages: 2.45 3.11 3.40
  ```

- 信息含义

  - 当前时间
  - 系统运行时间
  - 正在登录用户数
  - 过去 1 分钟、5 分钟、15 分钟的平均负载（Load Average）
    - 简单来说，平均负载是指单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数
      - 可运行状态的进程，是指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们常用 ps 命令看到的，处于 R 状态（Running 或 Runnable）的进程
      - 不可中断状态的进程则是正处于内核态关键流程中的进程，并且这些流程是不可打断的，比如最常见的是等待硬件设备的 I/O 响应，也就是我们在 ps 命令中看到的 D 状态（Uninterruptible Sleep，也称为 Disk Sleep）的进程
    - 但它实际上是活跃进程数的指数衰减平均值

- 注意
  - 平均负载高有可能是 CPU 密集型进程导致的
  - 平均负载高并不一定代表 CPU 使用率高，还有可能是 I/O 更繁忙了
  - 当发现负载高的时候，你可以使用 mpstat、pidstat 等工具，辅助分析负载的来源

## 上下文切换

- 进程上下文切换

  - 系统调用，完成从用户态到内核态的转变

    - CPU 寄存器里原来用户态的指令位置，需要先保存起来。接着，为了执行内核态代码
    - CPU 寄存器需要更新为内核态指令的新位置。最后才是跳转到内核态运行内核任务
    - 而系统调用结束后，CPU 寄存器需要恢复原来保存的用户态，然后再切换到用户空间，继续运行进程。所以，一次系统调用的过程，其实是发生了两次 CPU 上下文切换

  - 进程间切换

    - 从一个进程切换到另一个进程运行
    - 在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈
    - 进程的上下文不仅包括了虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的状态

- 线程上下文切换

  - 前后两个线程属于不同进程
    - 因为资源不共享，所以切换过程就跟进程上下文切换是一样
  - 前后两个线程属于同一个进程
    - 因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据

- 中断上下文切换

  - 中断上下文切换并不涉及到进程的用户态
    - 即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等
  - 上半部
    - 用来快速处理中断,主要处理跟硬件紧密相关的或时间敏感的工作
  - 下半部
    - 延迟处理上半部未完成的工作，通常以内核线程的方式运行；通过软中断信号通知

- 查看系统的上下文切换情况

  - 通过 vmstat 查看系统总体的上下文切换
  - 通过 pidstat 查看进程切换详情
    - cswch：自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换
    - nvcswch：非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换

- 问题分析

  - 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题
  - 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈
  - 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型

## 进程调度时机

- 为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行
- 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行
- 当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，自然也会重新调度
- 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行
- 发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序

## CPU 使用率

- CPU 使用率 = 1 - 空闲时间 / 总 CPU 时间
- 平均 CPU 使用率 = (空闲时间 2 - 空闲时间 1) / (总 CPU 时间 2 - 总 CPU 时间 1)
  - 性能工具一般都会取间隔一段时间（比如 3 秒）的两次值
- 常用的性能分析工具
  - top/htop 显示了系统总体的 CPU 和内存使用情况，以及各个进程的资源使用情况
  - ps 显示了每个进程的资源使用情况

## 工具

- stress Linux 系统压力测试工具

  - stress --cpu 1 --timeout 600
  - stress -i 1 --timeout 600
  - stress -c 8 --timeout 600

- sysbench 多线程的基准测试工具

  - sysbench --threads=10 --max-time=300 threads run

- sysstat 包含了常用的 Linux 性能工具

  - mpstat 是一个常用的多核 CPU 性能分析工具，用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标
    - mpstat -P ALL 5 20
  - pidstat 是一个常用的进程性能分析工具，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标
    - pidstat -u 5 1
    - pidstat -w 5
    - pidstat -wt 1

- vmstat 一个常用的系统性能分析工具，主要用来分析系统的内存使用情况，也常用来分析 CPU 上下文切换和中断的次数

  - 命令：vmstat 1 3
  - 信息解读
    - cs（context switch）是每秒上下文切换的次数
    - in（interrupt）则是每秒中断的次数
    - r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数
    - b（Blocked）则是处于不可中断睡眠状态的进程数

- iostat 动态监视系统的磁盘操作活动

```bash

# 显示所有设备负载情况
# iostat
Linux 4.14.0_1-0-0-48 	07/08/2022 	_x86_64_	(56 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.39    0.19    0.38    0.11    0.00   98.93

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda              15.44         9.48       176.87   71242014 1328555302

# 查看TPS和吞吐量
# iostat -d -x -k 1 1
Linux 4.14.0_1-0-0-48 	07/08/2022 	_x86_64_	(56 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     9.55    0.46   14.98     9.48   176.87    24.14     0.28   17.74  109.27   14.95   2.78   4.29

如果%iowait的值过高，表示硬盘存在I/O瓶颈。
如果 %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。
如果 svctm 比较接近 await，说明 I/O 几乎没有等待时间；
如果 await 远大于 svctm，说明I/O 队列太长，io响应太慢，则需要进行必要优化。
如果avgqu-sz比较大，也表示有大量io在等待

# 查看设备使用率（%util）和响应时间（await）
# iostat -d -x -k 1 1
Linux 4.14.0_1-0-0-48 	07/08/2022 	_x86_64_	(56 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     9.55    0.46   14.98     9.48   176.87    24.14     0.28   17.74  109.27   14.95   2.78   4.29
```

- perf

  - 以性能事件采样为基础，不仅可以分析系统的各种事件和内核性能，还可以用来分析指定应用程序的性能问题
  - 命令
    - perf top
    - perf record/report

- ab

  - apache bench, 常用的 HTTP 服务性能测试工具
  - ab -c 10 -n 100 your_url:port 并发 10 个请求测试 Nginx 性能，总共测试 100 个请求

- dstat

  - 系统资源使用分析，如磁盘

- pstack 跟踪进程栈空间

```bash
pstack 36897
```

- strace 跟踪进程执行时的系统调用和所接收的信号

```bash
# strace -p 36897

execve("/usr/bin/strace", ["strace", "-p", "36897"], 0x7fff22c9c440 /* 27 vars */) = 0
brk(NULL)                               = 0xc60000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc291aeb000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=43388, ...}) = 0
mmap(NULL, 43388, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fc291ae0000
close(3)                                = 0
open("/lib64/librt.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000\"\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=43712, ...}) = 0
mmap(NULL, 2128952, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc2916c3000
mprotect(0x7fc2916ca000, 2093056, PROT_NONE) = 0
mmap(0x7fc2918c9000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6000) = 0x7fc2918c9000
close(3)                                = 0
open("/lib64/libdw.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\237\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=338672, ...}) = 0
mmap(NULL, 2427184, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc291472000
mprotect(0x7fc2914c0000, 2097152, PROT_NONE) = 0
mmap(0x7fc2916c0000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4e000) = 0x7fc2916c0000
close(3)                                = 0
...

# 跟踪服务程序
strace -o output.txt -T -tt -e trace=all -p 36897
```

- pmap <PID>

  - 分析线程堆栈

```bash
pstrack 36897
```

- sar (可使用 vmstat, prstat 替换)
  - 查看 CPU 使用率： sar -u 1 2 (每秒采样 1 次，共采样 2 次)
  - 查看 CPU 平均负载： sar -q 1 2 (查看运行队列中的进程数、系统上的进程大小、平均负载等)
  - 查看内存使用： sar -r 1 2 / vmstat 1 3
  - 查看内存页交换查询： sar -W 1 3

## 网络

- netstat 用于显示各种网络相关信息，如网络连接，路由表，接口状态 (Interface Statistics)，masquerade 连接，多播成员 (Multicast Memberships) 等等

  - 列出所有端口 (包括监听和未监听的): netstat -a
  - 列出所有 tcp 端口: netstat -at
  - 列出所有有监听的服务状态: netstat -l
  - 端口查询: netstat -antp | grep 8080

  ```bash
  # netstat -antp | grep 8669
  tcp6       0      0 :::8669                 :::*                    LISTEN      36897/nebula-graphd

  # ps 36897
    PID TTY      STAT   TIME COMMAND
  36897 ?        Ssl    4:47 ./bin/nebula-graphd --flagfile ./nebula-graph/etc/nebula-graphd.conf
  ```

## IPCS

- IPCS 查询

```bash
# ipcs
IPC status from <running system> as of Fri Jul  8 15:58:10 CST 2022
T     ID     KEY        MODE       OWNER    GROUP
Message Queues:

T     ID     KEY        MODE       OWNER    GROUP
Shared Memory:

T     ID     KEY        MODE       OWNER    GROUP
Semaphores:
s 720896 0xe93c17d9 --ra-ra-ra-     work    staff
s 262146 0x624b15dc --ra-------     work    staff
s 262147 0x89600dce --ra-------     work    staff
s 262148 0xbf832208 --ra-------     work    staff
s  65541 0xccb76beb --ra-------     work    staff
s  65542 0xfb27a582 --ra-------     work    staff
s  65543 0x27a6455a --ra-------     work    staff
s  65544 0xb1b0e0cd --ra-------     work    staff
```

## 文件分析

- nm 显示关于指定 File 中符号的信息，文件可以是对象文件、可执行文件或对象文件库

```bash
zsh ➜ nm main
00000001000a6bf8 s _$f64.3eb0000000000000
00000001000a6c00 s _$f64.3f50624dd2f1a9fc
00000001000a6c08 s _$f64.3f847ae147ae147b
00000001000a6c10 s _$f64.3fd3333333333333
00000001000a6c18 s _$f64.3fe6666666666666
00000001000a6c20 s _$f64.3fee666666666666
00000001000a6c28 s _$f64.3ff199999999999a
00000001000a6c30 s _$f64.3ff3333333333333
00000001000a6c38 s _$f64.3ffb333333333333
00000001000a6c40 s _$f64.4059000000000000
00000001000a6c48 s _$f64.40c3880000000000
00000001000a6c50 s _$f64.40f0000000000000
00000001000a6c58 s _$f64.412e848000000000
00000001000a6c60 s _$f64.7ff0000000000000
00000001000a6c68 s _$f64.bfd3333333333333
00000001000a6c70 s _$f64.bfe62e42fefa39ef
                 U ___error
0000000100133e60 b __cgo_init
0000000100133e68 b __cgo_notify_runtime_init_done
0000000100133e70 b __cgo_thread_start
0000000100133e78 b __cgo_yield
                 U __exit
000000010005ad70 t __rt0_arm64_darwin
00000001000570d0 t _aeshashbody
0000000100162f30 s _block_size
00000001000570a0 t _callRet
                 U _clock_gettime
                 U _close
                 U _closedir
0000000100001ab0 t _cmpbody
0000000100057770 t _debugCall1024
...
...
000000010011d2a0 s _unicode/utf8.acceptRanges
0000000100120260 s _unicode/utf8.first
                 U _usleep
                 U _write
```

- ogjdump 工具用来显示二进制文件的信息

```bash
zsh ➜ objdump -d main

main:   file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000100001000 <_runtime.text>:
100001000: ff 20 47 6f  umlal2.4s       v31, v7, v7[0]
100001004: 20 62 75 69  ldpsw   x0, x24, [x17, #-88]
100001008: 6c 64 20 49  <unknown>
10000100c: 44 3a 20 22  <unknown>
100001010: 5a 46 58 50  adr     x26, #723146
100001014: 5f 33 4b 38  ldurb   wzr, [x26, #179]
100001018: 5f 6d 45 65  <unknown>
10000101c: 34 6f 48 64  <unknown>
100001020: 57 56 6a 4c  <unknown>
100001024: 2f 66 64 5f  <unknown>
100001028: 53 63 6c 66  <unknown>
10000102c: 34 6d 57 50  adr     x20, #716198
100001030: 6c 56 79 46  <unknown>
100001034: 36 32 4b 67  <unknown>
...
...
```

- readelf 与 objdump 类似，展示的信息具体

```bash
# readelf -all /usr/bin/make
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x40431d
  Start of program headers:          64 (bytes into file)
  Start of section headers:          180896 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000004c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002e8  000002e8
       0000000000000b88  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400e70  00000e70
       000000000000040f  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000401280  00001280
       00000000000000f6  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000401378  00001378
       0000000000000070  0000000000000000   A       6     1     8
```

- size 查看程序运行时各个段的实际内存占用

```bash
zsh ➜ size main
__TEXT	__DATA	__OBJC	others	dec	hex
688128	310624	0	4295554186	4296552938	1001831ea
```

- xxd 十六进制显示数据

```bash
zsh ➜ xxd main
00000000: cffa edfe 0c00 0001 0000 0000 0200 0000  ................
00000010: 0e00 0000 7009 0000 0400 2000 0000 0000  ....p..... .....
00000020: 1900 0000 4800 0000 5f5f 5041 4745 5a45  ....H...__PAGEZE
00000030: 524f 0000 0000 0000 0000 0000 0000 0000  RO..............
00000040: 0000 0000 0100 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 1900 0000 3801 0000  ............8...
00000070: 5f5f 5445 5854 0000 0000 0000 0000 0000  __TEXT..........
00000080: 0000 0000 0100 0000 0080 0a00 0000 0000  ................
00000090: 0000 0000 0000 0000 0080 0a00 0000 0000  ................
000000a0: 0700 0000 0500 0000 0300 0000 0000 0000  ................
000000b0: 5f5f 7465 7874 0000 0000 0000 0000 0000  __text..........
000000c0: 5f5f 5445 5854 0000 0000 0000 0000 0000  __TEXT..........
000000d0: 0010 0000 0100 0000 8070 0800 0000 0000  .........p......
000000e0: 0010 0000 0400 0000 0000 0000 0000 0000  ................
000000f0: 0004 0080 0000 0000 0000 0000 0000 0000  ................
00000100: 5f5f 7379 6d62 6f6c 5f73 7475 6231 0000  __symbol_stub1..
00000110: 5f5f 5445 5854 0000 0000 0000 0000 0000  __TEXT..........
00000120: 8080 0800 0100 0000 3402 0000 0000 0000  ........4.......
00000130: 8080 0800 0500 0000 0000 0000 0000 0000  ................
...
...
```
