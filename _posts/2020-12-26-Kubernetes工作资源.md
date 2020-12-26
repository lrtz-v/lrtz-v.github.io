---
title: Kubernetes工作资源
date: 2020-12-26 20:30:01
tags: [k8s, Kubernetes, kubernetes]
---

# Kubernetes 工作资源

## Pod

- 是 Kubernetes 项目中的最小编排单位
- 负责调度、网络、存储，以及安全、容器的 Linux Namespace 相关的属性
- Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume

### 组成

- infra（k8s.gcr.io/pause） 容器
  - 在 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起
  - 对于 Pod 里的容器 A 和容器 B 来说
    - 它们可以直接使用 localhost 进行通信；
    - 它们看到的网络设备跟 Infra 容器看到的完全一样；
    - 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
    - 其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
    - Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。
- 用户容器

### Pod 状态

- Pending：Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功
- Running：Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中
- Succeeded：Pod 里的所有容器都正常运行完毕，并且已经退出了
- Failed：Pod 里至少有一个容器以不正常的状态
- Unknown：异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题

### Volume

- Secret
  - 把 Pod 想要访问的加密数据，存放到 Etcd 中。然后，你就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret 里保存的信息了
- ConfigMap
  - 保存的是不需要加密的、应用所需的配置信息
- Downward API
  - 让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息
- ServiceAccountToken
  - 任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server
  - 每一个 Pod，都已经自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume

### 关键字段

- NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段

```yaml
apiVersion: v1
kind: Pod
---
spec:
nodeSelector:
  disktype: ssd
```

- NodeName：一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字
- HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容

```yaml
apiVersion: v1
kind: Pod
---
spec:
  hostAliases:
    - ip: "10.1.2.3"
      hostnames:
        - "foo.remote"
        - "bar.remote"
```

- shareProcessNamespace：共享 PID Namespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
    - name: nginx
      image: nginx
    - name: shell
      image: busybox
      stdin: true
      tty: true
```

- ImagePullPolicy：定义了镜像拉取的策略
- Lifecycle：在容器状态发生变化时触发一系列“钩子”

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
    - name: lifecycle-demo-container
      image: nginx
      lifecycle:
        postStart:
          exec:
            command:
              [
                "/bin/sh",
                "-c",
                "echo Hello from the postStart handler > /usr/share/message",
              ]
        preStop:
          exec:
            command: ["/usr/sbin/nginx", "-s", "quit"]
```

- livenessProbe：健康检查

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
    - name: liveness
      image: busybox
      args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
```

- restartPolicy：Pod 恢复机制
- initContainers：所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
    - image: geektime/sample:v2
      name: war
      command: ["cp", "/sample.war", "/app"]
      volumeMounts:
        - mountPath: /app
          name: app-volume
  containers:
    - image: geektime/tomcat:7.0
      name: tomcat
      command: ["sh", "-c", "/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
      volumeMounts:
        - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
          name: app-volume
      ports:
        - containerPort: 8080
          hostPort: 8001
  volumes:
    - name: app-volume
      emptyDir: {}
```

- resources：限制容器资源
  - request：资源最小申请量
  - limits：资源申请上限

## Deployment Controller 控制器模式

### ReplicaSet

- ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数
- 支持 Label Selector
- 通过调整 Pod 数量，实现扩容/缩容
- 通过调整 Pod 镜像版本，实现滚动升级

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
```

### Deployment

- 实现 Pod 的“水平扩展 / 收缩”（horizontal scaling out/in），依赖 ReplicaSet
- 控制关系：Deployment -> ReplicaSet -> Pods
- 水平扩展 / 收缩
  - Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以了
  - kubectl scale deployment nginx-deployment --replicas=4
- 滚动更新
  - 将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程
  - 可以通过 RollingUpdateStrategy 配置一次“滚动”中创建/删除的 Pod 量
  - 通过 kubectl rollout undo 回滚操作
  - 修改过程
    - kubectl rollout pause 暂停 Deployment
    - kubectl edit / kubectl set image 修改 Deployment
    - kubectl rollout resume 恢复 Deployment
  - 设置 rollingUpdate 的 partition 控制 Pod 更新范围
- Deployment 认为 Pod 应该是一样的“无状态应用”（Stateless Application），没有启动顺序，但是实际使用场景中存在容器依赖的情况，实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

## Service

- Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制

- Service 又是如何被访问的
  - 以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式
  - 以 Service 的 DNS 方式
    - Normal Service
      - 通过 DNS 解析获取，拿到 Service 的 VIP
    - Headless Service
      - 以 DNS 记录的方式解析出被代理 Pod 的 IP 地址，不需要分配一个 VIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None  // 这个 Service，没有一个 VIP 作为“头”, 这个 Service 被创建后并不会被分配一个 VIP，它所代理的 Pod是采用Label Selector 机制选择出来的
  selector:
    app: nginx
```

## StatefulSet

- 在 Deployment 的基础上，扩展出了对“有状态应用”的初步支持。这个编排功能，就是：StatefulSet
- StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态
- StatefulSet 把真实世界里的应用状态，抽象为了两种情况

  - 拓扑状态
    - 服务之间存在启动顺序与依赖关系
    - 使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作
    - 通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口
  - 存储状态
    - 应用的多个实例分别绑定了不同的存储数据， Pod 的重启不影响数据一致
    - PVC
      - 使用方式
        - 定义一个 PVC，声明想要的 Volume 的属性
        - 在应用的 Pod 中，声明使用这个 PVC
    - PV
      - 根据 PVC 分配 Volume

- 应用管理

  - StatefulSet 的控制器直接管理的是 Pod
    - 每个 Pod 的 hostname、名字等都是不同的、携带了编号的，并且按照编号顺序逐一完成创建工作。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号
  - Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录
  - StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC

- 拓扑状态示例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  // 与deployment模式的区别，告诉 StatefulSet 控制器使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
            - containerPort: 80
              name: web
```

- 存储状态示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
        - name: rbdpd
          mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
          - "10.16.154.78:6789"  // 这里会暴露服务地址等信息，所以k8s增加了PVC和PV降低用户声明和使用持久化 Volume 的门槛，示例如下
          - "10.16.154.82:6789"
          - "10.16.154.83:6789"
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

- 第一步：定义 PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- 第二步：在应用的 Pod 中，声明使用这个 PVC
  - Volume 来自于由运维人员维护的 PV（Persistent Volume）对象

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim  // 指定 PVC 的名字，而完全不必关心 Volume 本身的定义
```

- PV 对象的 YAML 文件示例

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      # 使用 kubectl get pods -n rook-ceph 查看 rook-ceph-mon- 开头的 POD IP 即可得下面的列表
      - "10.16.154.78:6789"
      - "10.16.154.82:6789"
      - "10.16.154.83:6789"
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
```

- PVC、PV 的设计，使得 StatefulSet 对存储状态的管理成为了可能
  - Pod 被删除后，Volume 内的文件不会变

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

## DaemonSet

- 运行 DaemonSet Pod
  - 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上
  - 每个节点上只有一个这样的 Pod 实例
  - 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉
- 使用场景
  - 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络
  - 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录
  - 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集
- 如何保证每个 Node 上有且只有一个被管理的 Pod 呢
  - Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。检查当前这个 Node 上是不是有一个携带了 DaemonSet 指定标签的 Pod 在运行
    - 如果没有，则需要创建
    - 如果数量大于 1，则需要删除多余的 Pod
    - 只有一个，则为正常
- DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration 这两个调度器的小功能，保证了每个节点上有且只有一个 Pod

## Job Controller

- 并行作业的控制方法

  - spec.parallelism，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行
  - spec.completions，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数

- Job Controller 的工作原理

  - Job Controller 控制的对象，直接就是 Pod
  - 根据实际在 Running 状态 Pod 的数目、已经成功退出的 Pod 的数目，以及 parallelism、completions 参数的值共同计算出在这个周期里，应该创建或者删除的 Pod 数目，然后调用 Kubernetes API 来执行这个操作

- CronJob

  - 通过 schedule 配置 Cron 表达式
  - spec.concurrencyPolicy 配置 Job 执行时间过长，导致下一个创建 Pod 的周期到达的处理策略

## 容器网络

### 网络栈

- 网卡（Network Interface）
- 回环设备（Loopback Device）
- 路由表（Routing Table）
- iptables 规则

### 容器网络的实现原理

- 网桥（Bridge）

  - 在 Linux 中，能够起到虚拟交换机作用的网络设备
  - 工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上
  - Docker 项目会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信

- Veth Pair

  - 它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里
  - 这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的“网线”

- 单节点容器间通信

  - 容器 1 通过 eth0 网卡发送一个 ARP 广播，来通过 IP 地址查找对应的 MAC 地址
  - docker0 网桥就会扮演二层交换机的角色，把 ARP 广播转发到其他被“插”在 docker0 上的虚拟网卡上
  - 同样连接在 docker0 上的容器 2 的网络协议栈就会收到这个 ARP 请求，从而将 IP 所对应的 MAC 地址回复给容器 1
  - 有了这个目的 MAC 地址，容器 1 的 eth0 网卡就可以将数据包发出去；数据包会经过 docker0 转发到容器 2

- 当一个容器试图连接到另外一个宿主机时

  - 容器 1 发出的请求数据包，首先经过 docker0 网桥出现在宿主机上
  - 然后根据宿主机的路由表里的直连路由规则，对另一个宿主机的访问请求就会交给宿主机的 eth0 处理
  - 这个数据包就会经宿主机的 eth0 网卡转发到宿主机网络上，最终到达指定的宿主机上

- 跨主通信
  - Overlay Network（覆盖网络）
    - 我们需要在已有的宿主机网络上，再通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络

### Flannel - CoreOS 公司主推的容器网络方案

#### UDP 模式

- IP 包从容器经过 docker0 出现在宿主机

  - flannel0：它是一个 TUN 设备（Tunnel 设备）；
  - TUN 设备：在操作系统内核和用户应用程序之间传递 IP 包

- 然后又根据路由表进入 flannel0 设备后，宿主机上的 flanneld 进程（Flannel 项目在每个宿主机上的主进程），就会收到这个 IP 包

- 然后，flanneld 看到了这个 IP 包的目的地址，就把它发送给了 Node 2 宿主机

  - 在由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”
  - 子网与宿主机的对应关系，正是保存在 Etcd 当中
  - flanneld 进程可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24）
  - flanneld 在收到 container-1 发给 container-2 的 IP 包之后，就会把这个 IP 包直接封装在一个 UDP 包里，然后发送给 Node 2
  - Node 2 的 flanneld 收到数据包之后，会将包传给 docker0，再到容器 2

- 缺点
  - Flannel 进行 UDP 封装（Encapsulation）和解封装（Decapsulation）的过程，也都是在用户态完成的。在 Linux 操作系统中，上述这些上下文切换和用户态操作的代价其实是比较高的

#### VXLAN 模式

- 即 Virtual Extensible LAN（虚拟可扩展局域网）；是 Linux 内核本身就支持的一种网络虚似化技术。VXLAN 可以完全在内核态实现上述封装和解封装的工作

- VTEP 设备：VXLAN Tunnel End Point（虚拟隧道端点）
  - 它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的
  - VTEP 设备之间，就需要想办法组成一个虚拟的二层网络，即：通过二层数据帧进行通信
  - “源 VTEP 设备”收到“原始 IP 包”后，就要想办法把“原始 IP 包”加上一个目的 MAC 地址，封装成一个二层数据帧，然后发送给“目的 VTEP 设备”
    - Flannel 会在每台节点启动时把它的 VTEP 设备对应的 ARP 记录，直接下放到其他每台宿主机上
  - 然后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去
  - 这个 UDP 包该发给哪台宿主机呢
    - flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发
    - 在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库
    - 这个 flannel.1“网桥”对应的 FDB 信息，也是 flanneld 进程负责维护的，可以使用“目的 VTEP 设备”的 MAC 地址进行查询目标设备 IP

#### host-gw 模式

- 将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址
- “主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）
- 免除了额外的封包和解包带来的性能损耗

### Kubernetes 网络模型与 CNI 网络插件

- Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0
- CNI 网桥只是接管所有 CNI 插件负责的、即 Kubernetes 创建的容器（Pod）
- 使用 CNI 的原因
  - Kubernetes 项目并没有使用 Docker 的网络模型（CNM），所以它并不希望、也不具备配置 docker0 网桥的能力
  - 另一方面，这还与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关
    - Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈
- CNI 的基础可执行文件
  - 第一类，Main 插件，它是用来创建具体网络设备的二进制文件
    - 比如，bridge（网桥设备）、ipvlan、loopback（lo 设备）、macvlan、ptp（Veth Pair 设备），以及 vlan
  - 第二类，IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件
    - 比如，dhcp，这个文件会向 DHCP 服务器发起请求；host-local，则会使用预先配置的 IP 地址段来进行分配
  - 第三类，是由 CNI 社区维护的内置 CNI 插件
    - 比如：flannel，就是专门为 Flannel 项目提供的 CNI 插件；tuning，是一个通过 sysctl 调整网络设备参数的二进制文件；portmap，是一个通过 iptables 配置端口映射的二进制文件；bandwidth，是一个使用 Token Bucket Filter (TBF) 来进行限流的二进制文件

#### 以 Flannel 项目为例

- 要实现一个给 Kubernetes 用的容器网络方案，其实需要做两部分工作
  - 首先，实现这个网络方案本身
    - 实现 flanneld 进程里的主要逻辑。比如，创建和配置 flannel.1 设备、配置宿主机路由、配置 ARP 和 FDB 表里的信息等
  - 然后，实现该网络方案对应的 CNI 插件
    - 配置 Infra 容器里面的网络栈，并把它连接在 CNI 网桥上

// TODO: Kubernetes 的资源模型与资源管理
