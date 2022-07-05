---
template: main.html
tags:
  - K8S
---
# Kubernetes 基础

## Kubernetes 组件

### 控制层组件

- kube-apiserver
  - 提供 HTTTP Rest 接口
- etcd
  - 一致性和高可用性的键值数据库，保存 Kubernetes 所有集群数据的后台数据库
- kube-scheduler
  - 负责资源调度
- kube-controller-manager

  - 在主节点上运行控制器的组件，每个控制器都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行
  - 控制器包括
    - 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
    - 副本控制器（Replication Controller）: 负责为系统中的每个副本控制器对象维护正确数量的 Pod
    - 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
    - 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

- cloud-controller-manager
  - 嵌入特定云的控制逻辑的控制层组件
  - 控制器包括
    - 节点控制器（Node Controller）: 用于在节点终止响应后检查云提供商以确定节点是否已被删除
    - 路由控制器（Route Controller）: 用于在底层云基础架构中设置路由
    - 服务控制器（Service Controller）: 用于创建、更新和删除云提供商负载均衡器

### Node 组件

- 作用：在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行环境
- kubelet
  - 一个在集群中每个节点上运行的代理。 它保证容器都运行在 Pod 中；
  - kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康
- kube-proxy
  - 每个节点上运行的网络代理，维护节点上的网络规则
- Container Runtime
  - 负责运行容器
  - Kubernetes 支持多个容器运行环境: Docker、 containerd、CRI-O 以及任何实现 Kubernetes CRI (容器运行环境接口)的容器环境
- 插件（Addons）
  - 使用 Kubernetes 资源（DaemonSet、 Deployment 等）实现集群功能。 因为这些插件提供集群级别的功能，插件中命名空间域的资源属于 kube-system 命名空间
  - 常见插件如下
    - DNS
    - 仪表盘
    - 容器资源监控
    - 集群层面日志

## Kubernetes 对象

### 理解 Kubernetes 对象

- Kubernetes 对象 是持久化的实体使用这些实体去表示整个集群的状态，它们描述了如下信息
  - 哪些容器化应用在运行（以及在哪些节点上）
  - 可以被应用使用的资源
  - 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略
- 对象规约（Spec）与状态（Status）

  - spec：描述你希望对象所具有的特征： 期望状态（Desired State），如需要有 3 个副本运行
  - status：描述了对象的 当前状态（Current State），由 Kubernetes 系统和组件 设置并更新

- 描述 Kubernetes 对象

  ```yaml
  application/deployment.yaml

  apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    selector:
      matchLabels:
        app: nginx
    replicas: 2 # tells deployment to run 2 pods matching the template
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
          - containerPort: 80
  ```

  - 使用姿势：kubectl apply -f application/deployment.yaml --record

- 必需字段
  - apiVersion - 创建该对象所使用的 Kubernetes API 的版本
  - kind - 想要创建的对象的类别
  - metadata - 帮助唯一性标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace
  - spec

### Kubernetes 对象管理

- 管理技巧
  - 命令式
    - 用户将操作传给 kubectl 命令作为参数或标志
    - 例如：kubectl run nginx --image nginx
  - 命令式对象配置
    - kubectl 命令指定操作（创建，替换等），可选标志和至少一个文件名。指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义
    - 例如：kubectl apply -f application/deployment.yaml --record
  - 声明式对象配置
    - 配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作
    - 例如：kubectl apply -f configs/

### 对象名称和 IDs

- 名称
  - 集群中的每一个对象都有一个名称 来标识在同类资源中的唯一性
  - 客户端提供的字符串，引用资源 url 中的对象，如/api/v1/pods/some name
- UID
  - Kubernetes 系统生成的字符串，唯一标识对象

### 命名空间

- Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为命名空间
- 命名空间为名称提供了一个范围。资源的名称需要在命名空间内是唯一的，但不能跨命名空间，也不能相互嵌套
- 初始命名空间
  - default 没有指明使用其它命名空间的对象所使用的默认命名空间
  - kube-system Kubernetes 系统创建对象所使用的命名空间
  - kube-public 这个命名空间是自动创建的，所有用户（包括未经过身份验证的用户）都可以读取它。 这个命名空间主要用于集群使用，以防某些资源在整个集群中应该是可见和可读的。 这个命名空间的公共方面只是一种约定，而不是要求。
  - kube-node-lease 此命名空间用于与各个节点相关的租期（Lease）对象； 此对象的设计使得集群规模很大时节点心跳检测性能得到提升
- 使用
  - 查看
    - kubectl get namespace
  - 设置
    - kubectl run nginx --image=nginx --namespace=<命名空间名称>
    - kubectl get pods --namespace=<命名空间名称>
  - 设置命名空间偏好----永久保存命名空间，以用于对应上下文中所有后续 kubectl 命令
    - kubectl config set-context --current --namespace=<命名空间名称>
- 命名空间和 DNS
  - 创建一个 Service 时， Kubernetes 会创建一个相应的 DNS 条目
    - 该条目的形式是 <服务名称>.<命名空间名称>.svc.cluster.local，这意味着如果容器只使用 <服务名称>，它将被解析到本地命名空间的服务
- 并非所有对象都在命名空间中
  - 查看哪些 Kubernetes 资源在命名空间中，哪些不在命名空间中
    - kubectl api-resources --namespaced=true
    - kubectl api-resources --namespaced=false

### Label 与 Selector

- 标签（Labels） 是附加到 Kubernetes 对象（比如 Pods）上的键值对，用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
  name: annotations-demo
  labels:
    env: dev
  spec:
  containers:
    - name: nginx
      image: nginx:1.7.9
      ports:
        - containerPort: 80
  ```

- Selector

  - 通过 Selector，用户可以识别一组对象
  - 基于等值的选择器
    - 运算符有=、== 和 !=
    - 如 env=dev
  - 基于集合的选择器
    - 操作符：in、notin 和 exists
    - 如 env in (dev, qa)

- API

  - kubectl get pods -l env=dev
  - kubectl get pods -l 'env in (production, qa),tier in (frontend)'

- 推荐使用的标签

| 键                           | 描述                                               | 示例         | 类型   |
| ---------------------------- | -------------------------------------------------- | ------------ | ------ |
| app.kubernetes.io/name       | 应用程序的名称                                     | mysql        | 字符串 |
| app.kubernetes.io/instance   | 用于唯一确定应用实例的名称                         | mysql-abcxzy | 字符串 |
| app.kubernetes.io/version    | 应用程序的当前版本（例如，语义版本，修订版哈希等） | 5.7.21       | 字符串 |
| app.kubernetes.io/component  | 架构中的组件                                       | database     | 字符串 |
| app.kubernetes.io/part-of    | 此级别的更高级别应用程序的名称                     | wordpress    | 字符串 |
| app.kubernetes.io/managed-by | 用于管理应用程序的工具                             | helm         | 字符串 |

### 注解

- 为对象附加任意的非标识的元数据

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
  spec:
  containers:
    - name: nginx
      image: nginx:1.7.9
      ports:
        - containerPort: 80
  ```

### 字段选择器（Field selectors）

- 允许你根据一个或多个资源字段的值 筛选 Kubernetes 资源，例如

  - metadata.name=my-service
  - metadata.namespace!=default
  - status.phase=Pending
  - kubectl get pods --field-selector status.phase=Running

- 支持的字段
  - 所有资源类型都支持 metadata.name 和 metadata.namespace 字段
- 支持的操作符
  - =、==和 !=
- 链式选择器
  - kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
- 多种资源类型
  - kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default

## Kubernetes 架构

### 节点

#### 概念

- 节点可以是一个虚拟机或者物理机器，取决于所在的集群配置
- 每个节点都包含用于运行 Pod（容器运行于 Pod 中） 所需要的服务，这些服务由 控制层负责管理
- 节点上的组件包括 kubelet、 Container Runtime 以及 kube-proxy

#### 管理

- 向 API Server 添加节点的方式主要有两种

  - 节点上的 kubelet 向控制面执行自注册
  - 手动注册

- 例如使用下面的 JSON 对象来创建 Node 对象

  ```json
  {
    "kind": "Node",
    "apiVersion": "v1",
    "metadata": {
      "name": "10.240.79.157",
      "labels": {
        "name": "my-first-k8s-node"
      }
    }
  }
  ```

  - Kubernetes 会在内部创建一个 Node 对象作为节点的表示。Kubernetes 检查 kubelet 向 API 服务器注册节点时使用的 metadata.name 字段是否匹配

- 节点自注册

  - 当 kubelet 标志 --register-node 为 true（默认）时，它会尝试向 API 服务注册自己
  - 对于自注册模式，kubelet 使用下列参数启动
    - --kubeconfig - 用于向 API 服务器表明身份的凭据路径
    - --cloud-provider - 与某云驱动 进行通信以读取与自身相关的元数据的方式
    - --register-node - 自动向 API 服务注册
    - --register-with-taints - 使用所给的污点列表（逗号分隔的 <key>=<value>:<effect>）注册节点。 当 register-node 为 false 时无效
    - --node-ip - 节点 IP 地址
    - --node-labels - 在集群中注册节点时要添加的 标签。 （参见 NodeRestriction 准入控制插件所实施的标签限制）
    - --node-status-update-frequency - 指定 kubelet 向控制面发送状态的频率

- 手动节点管理

  - 手动创建节点对象时，设置 kubelet 标志 --register-node=false
  - kubectl cordon 会将节点标记为“不可调度（Unschedulable）

- 节点状态

  - 个节点的状态包含以下信息，查询命令：kubectl describe node <节点名称>

    - 地址

      - HostName：由节点的内核设置。可以通过 kubelet 的 --hostname-override 参数覆盖。
      - ExternalIP：通常是节点的可外部路由（从集群外可访问）的 IP 地址。
      - InternalIP：通常是节点的仅可在集群内部路由的 IP 地址。

    - 状况（conditions）：描述了所有 Running 节点的状态
    - 容量与可分配
    - 信息

### 控制层到节点通信

- 节点到控制层
  - 想要连接到 apiserver 的 Pod 可以使用服务账号安全地进行连接。 当 Pod 被实例化时，Kubernetes 自动把公共根证书和一个有效的持有者令牌注入到 Pod 里
  - kubernetes 服务（位于所有命名空间中）配置了一个虚拟 IP 地址，用于（通过 kube-proxy）转发 请求到 apiserver 的 HTTPS 末端
- 控制层到节点
  - 有两种主要的通信路径
    - 从 apiserver 到集群中每个节点上运行的 kubelet 进程
      - 用于
        - 获取 Pod 日志
        - 挂接（通过 kubectl）到运行中的 Pod
        - 提供 kubelet 的端口转发功能
    - 从 apiserver 通过它的代理功能连接到任何节点、Pod 或者服务
      - 默认为纯 HTTP 方式，因此既没有认证，也没有加密
  - 其他方式
    - SSH 隧道（目前已被废弃）
      - apiserver 建立一个到集群中各节点的 SSH 隧道（连接到在 22 端口监听的 SSH 服务） 并通过这个隧道传输所有到 kubelet、节点、Pod 或服务的请求
    - Konnectivity 服务
      - Konnectivity 服务提供 TCP 层的代理，以便支持从控制面到集群的通信

### 控制器

- 在 Kubernetes 中，控制器通过监控集群 的公共状态，并致力于将当前状态转变为期望的状态

#### 控制器模式

- 控制器模式

  - 通过 API 服务器来控制
    - Deployment 控制器和 Job 控制器是 Kubernetes 内置控制器的典型例子， 内置控制器通过和集群 API 服务器交互来管理状态
  - 直接控制
    - 相比 Job 控制器，有些控制器需要对集群外的一些东西进行修改，和外部状态交互的控制器从 API 服务器获取到它想要的状态，然后直接和外部系统进行通信 并使当前状态更接近期望状态

- 运行控制器的方式
  - Kubernetes 内置一组控制器，运行在 kube-controller-manager 内。 这些内置的控制器提供了重要的核心功能
  - Deployment 控制器和 Job 控制器是 Kubernetes 内置控制器的典型例子

### [云控制器管理器](https://kubernetes.io/zh/docs/concepts/architecture/cloud-controller/)

// TODO

## 容器

### 镜像

- 镜像拉取策略（IfNotPresent）
  - 设置容器的 imagePullPolicy 为 Always
  - 省略 imagePullPolicy，并使用 :latest 作为要使用的镜像的标签
  - 省略 imagePullPolicy 和要使用的镜像标签
  - 启用 AlwaysPullImages 准入控制器（Admission Controller）
- 使用私有仓库

  - 配置 Node 对私有仓库认证

    - Docker 将私有仓库的密钥保存在 $HOME/.dockercfg 或 $HOME/.docker/config.json 文件中，kubelet 会在拉取镜像时将其用作凭据数据来源

      - 创建使用私有镜像的 Pod 来验证，例如

        ```bash
        kubectl apply -f - <<EOF
        apiVersion: v1
        kind: Pod
        metadata:
          name: private-image-test-1
        spec:
          containers:
            - name: uses-private-image
              image: $PRIVATE_IMAGE_NAME
              imagePullPolicy: Always
              command: [ "echo", "SUCCESS" ]
        EOF
        ```

  - 使用 Docker Config 创建 Secret

  ```bash
  kubectl create secret docker-registry <名称> \
  --docker-server=DOCKER_REGISTRY_SERVER \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASSWORD \
  --docker-email=DOCKER_EMAIL
  ```

  - 在 Pod 中引用 ImagePullSecrets

  ```bash
  cat <<EOF > pod.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: foo
    namespace: awesomeapps
  spec:
    containers:
      - name: foo
        image: janedoe/awesomeapp:v1
    imagePullSecrets:
      - name: myregistrykey
  EOF

  cat <<EOF >> ./kustomization.yaml
  resources:
  - pod.yaml
  EOF
  ```

### 容器环境

- 容器环境给容器提供了几个重要的资源
  - 文件系统，其中包含一个镜像 和一个或多个的卷
  - 容器自身的信息
  - 集群中其他对象的信息

### 容器运行时类(Runtime Class)

- RuntimeClass 是一个用于选择容器运行时配置的特性，容器运行时配置用于运行 Pod 中的容器

  - 可以在不同的 Pod 设置不同的 RuntimeClass，以提供性能与安全性之间的平衡。 例如，如果你的部分工作负载需要高级别的信息安全保证，你可以决定在调度这些 Pod 时尽量使它们在使用硬件虚拟化的容器运行时中运行

- 设置

  - 在节点上配置 CRI 实现
    - RuntimeClass 的配置依赖于 运行时接口（CRI）的实现，所有这些配置都具有相应的 handler 名，并被 RuntimeClass 引用
  - 创建相应的 RuntimeClass 资源

    - 定义

    ```yaml
    apiVersion: node.k8s.io/v1beta1 # RuntimeClass 定义于 node.k8s.io API 组
    kind: RuntimeClass
    metadata:
      name: myclass # 用来引用 RuntimeClass 的名字
      # RuntimeClass 是一个集群层面的资源
    handler: myconfiguration # 对应的 CRI 配置的名称
    ```

- 使用说明

  - 在 Pod spec 中指定 runtimeClassName 即可，这一设置会告诉 kubelet 使用所指的 RuntimeClass 来运行该 pod

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    runtimeClassName: myclass
    # ...
  ```

  - [CRI 配置](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/)

### 容器生命周期回调

- 容器回调

  - PostStart
    - 这个回调在容器被创建之后立即被执行。 但是，不能保证回调会在容器入口点（ENTRYPOINT）之前执行。 没有参数传递给处理程序
  - PreStop
    - 在容器因 API 请求或者管理事件（诸如存活态探针失败、资源抢占、资源竞争等）而被终止之前， 此回调会被调用
    - 如果容器已经处于终止或者完成状态，则对 preStop 回调的调用将失败
    - 此调用是阻塞的，也是同步调用，因此必须在发出删除容器的信号之前完成

- 回调处理程序的实现

  - Exec - 在容器的 cgroups 和名称空间中执行特定的命令（例如 pre-stop.sh）。 命令所消耗的资源计入容器的资源消耗。
  - HTTP - 对容器上的特定端点执行 HTTP 请求。

- 回调处理程序执行

  - Kubernetes 管理系统根据回调动作执行其处理程序， exec 和 tcpSocket 在容器内执行，而 httpGet 则由 kubelet 进程执行
  - 回调处理程序尽可能的轻量级，需要考虑长时间运行的情况

- 回调递送保证

  - 回调的递送应该是 至少一次，这意味着对于任何给定的事件， 例如 PostStart 或 PreStop，回调可以被调用多次。 如何正确处理被多次调用的情况，是回调实现所要考虑的问题

- 调试回调处理程序
  - 回调处理程序的日志不会在 Pod 事件中公开。 如果处理程序由于某种原因失败，它将添加一个事件

## 工作负载

### Pods

- Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元；是一组（一个或多个） 容器，这些容器共享存储、网络、以及怎样运行这些容器的声明
  - Pod 的共享上下文包括一组 Linux 名字空间、控制组（cgroup）和可能一些其他的隔离 方面，即用来隔离 Docker 容器的技术
- Pod 中的内容总是并置（colocated）的并且一同调度，在共享的上下文中运行
- 除了应用容器，Pod 还可以包含在 Pod 启动期间运行的 Init 容器，Init 容器会在启动应用容器之前运行并完成

#### 使用 Pod

- 不需要直接创建 Pod，甚至单实例 Pod， 而是会使用诸如 Deployment 或 Job 这类工作负载资源 来创建 Pod。如果 Pod 需要跟踪状态，可以考虑 StatefulSet 资源
- Kubernetes 集群中的 Pod 主要有两种用法

  - 运行单个容器的 Pod
    - 每个 Pod 一个容器"模型是最常见的 Kubernetes 用例，Kubernetes 直接管理 Pod，而不是容器
  - 运行多个协同工作的容器的 Pod
    - Pod 可能封装由多个紧密耦合且需要共享资源的共处容器组成的应用程序

- Pod 模版

  - Pod 模板是包含在工作负载对象中的规范，用来创建 Pod。这类负载资源包括 Deployment、 Job 和 DaemonSets 等
  - 工作负载的控制器会使用负载对象中的 PodTemplate 来生成实际的 Pod

  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: hello
  spec:
    template:
      # 这里是 Pod 模版
      spec:
        containers:
          - name: hello
            image: busybox
            command: ["sh", "-c", 'echo "Hello, Kubernetes!" && sleep 3600']
        restartPolicy: OnFailure
      # 以上为 Pod 模版
  ```

  - 资源共享和通信
    - Pod 使它的成员容器间能够进行数据共享和通信
  - Pod 中的存储
    - 一个 Pod 可以设置一组共享的存储卷
  - Pod 联网
    - 每个 Pod 都在每个地址族中获得一个唯一的 IP 地址。 Pod 中的每个容器共享网络名字空间，包括 IP 地址和网络端口
    - Pod 内 的容器可以使用 localhost 互相通信。 当 Pod 中的容器与 Pod 之外 的实体通信时，它们必须协调如何使用共享的网络资源 （例如端口）

#### 静态 Pod

- 静态 Pod 通常绑定到某个节点上的 kubelet
- kubelet 自动尝试为每个静态 Pod 在 Kubernetes API 服务器上创建一个 镜像 Pod。 这意味着在节点上运行的 Pod 在 API 服务器上是可见的，但不可以通过 API 服务器来控制

#### Pod 怎样管理多个容器

- Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身

#### Pod 的生命周期

- Pod 阶段
  - PodStatus.phase
    - Pending（悬决） Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间，
    - Running（运行中） Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
    - Succeeded（成功） Pod 中的所有容器都已成功终止，并且不会再重启。
    - Failed（失败） Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。
    - Unknown（未知） 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。
- 容器状态
  - Waiting （等待），仍在运行它完成启动所需要的操作
  - Running（运行中），表明容器正在执行状态并且没有问题发生
  - Terminated（已终止），开始执行并且或者正常结束或者因为某些原因失败
- 容器重启策略
  - spec.restartPolicy：Always、OnFailure 和 Never
- Pod 状况

  - PodStatus.PodConditions
    - PodScheduled：Pod 已经被调度到某节点；
    - ContainersReady：Pod 中所有容器都已就绪；
    - Initialized：所有的 Init 容器 都已成功启动；
    - Ready：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中。
  - Pod 就绪态

    - 通过 spec.readinessGates 为 kubelet 提供一组额外的状况
    - 就绪态基于 Pod 的 status.conditions 字段的当前值来做决定

    ```yaml
    kind: Pod
    ---
    spec:
      readinessGates:
        - conditionType: "www.example.com/feature-1"
    status:
      conditions:
        - type: Ready # 内置的 Pod 状况
          status: "False"
          lastProbeTime: null
          lastTransitionTime: 2018-01-01T00:00:00Z
        - type: "www.example.com/feature-1" # 额外的 Pod 状况
          status: "False"
          lastProbeTime: null
          lastTransitionTime: 2018-01-01T00:00:00Z
      containerStatuses:
        - containerID: docker://abcd...
          ready: true
    ```

- 容器探针

  - Probe 是由 kubelet 对容器执行的定期诊断；要执行诊断，kubelet 调用由容器实现的 Handler （处理程序）。有三种类型的处理程序
    - ExecAction： 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
    - TCPSocketAction： 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。
    - HTTPGetAction： 对容器的 IP 地址上指定端口和路径执行 HTTP Get 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。
  - 每次探测都将获得以下三种结果之一：
    - Success（成功）：容器通过了诊断。
    - Failure（失败）：容器未通过诊断。
    - Unknown（未知）：诊断失败，因此不会采取任何行动。
  - 探针类型
    - livenessProbe：指示容器是否正在运行
    - readinessProbe：指示容器是否准备好为请求提供服务
    - startupProbe: 指示容器中的应用是否已经启动

- Pod 的终止
  - 体面的终止
    - 容器运行时会发送一个 TERM 信号到每个容器中的主进程。 很多容器运行时都能够注意到容器镜像中 STOPSIGNAL 的值，并发送该信号而不是 TERM
    - 一旦超出了体面终止限期，容器运行时会向所有剩余进程发送 KILL 信号，之后 Pod 就会被从 API 服务器 上移除
    - 例如
      - 使用 kubectl 工具手动删除某个特定的 Pod，而该 Pod 的体面终止限期是默认值（30 秒）
      - API 服务器中的 Pod 对象被更新，记录涵盖体面终止限期在内 Pod 的最终死期，超出所计算时间点则认为 Pod 已死（dead）；describe 信息中 Pod 会显示为 "Terminating" （正在终止）
        - 如果 Pod 中的容器之一定义了 preStop 回调， kubelet 开始在容器内运行该回调逻辑。如果超出体面终止限期时，preStop 回调逻辑 仍在运行，kubelet 会请求给予该 Pod 的宽限期一次性增加 2 秒钟；如果 preStop 回调所需要的时间长于默认的体面终止限期，必须修改 terminationGracePeriodSeconds 属性值来使其正常工作
        - kubelet 接下来触发容器运行时发送 TERM 信号给每个容器中的进程 1
      - 与此同时，kubelet 启动体面关闭逻辑，控制面会将 Pod 从对应的端点列表（以及端点切片列表， 如果启用了的话）中移除；ReplicaSets 和其他工作负载资源 不再将关闭进程中的 Pod 视为合法的、能够提供服务的副本。关闭动作很慢的 Pod 也无法继续处理请求数据，因为负载均衡器（例如服务代理）已经在终止宽限期开始的时候 将其从端点列表中移除
      - 超出终止宽限期限时，kubelet 会触发强制关闭过程。容器运行时会向 Pod 中所有容器内 仍在运行的进程发送 SIGKILL 信号。 kubelet 也会清理隐藏的 pause 容器，如果容器运行时使用了这种容器的话
      - kubelet 触发强制从 API 服务器上删除 Pod 对象的逻辑，并将体面终止限期设置为 0 （这意味着马上删除）
      - API 服务器删除 Pod 的 API 对象，从任何客户端都无法再看到该对象
  - 强制终止
    - kubectl delete 命令支持 --grace-period=<seconds> 选项，允许你重载宽限期限默认值， 设定自己希望的期限值
    - 将宽限期限强制设置为 0 意味着立即从 API 服务器删除 Pod。 如果 Pod 仍然运行于某节点上，强制删除操作会触发 kubelet 立即执行清理操作；必须在设置 --grace-period=0 的同时额外设置 --force 参数才能发起强制删除请求
    - 执行强制删除操作时，API 服务器不再等待来自 kubelet 的、关于 Pod 已经在原来运行的节点上终止执行的确认消息。 API 服务器直接删除 Pod 对象，这样新的与之同名的 Pod 即可以被创建。 在节点侧，被设置为立即终止的 Pod 仍然会在被强行杀死之前获得一点点的宽限时间
  - 失效 Pod 的垃圾收集
    - 对于已失败的 Pod 而言，对应的 API 对象仍然会保留在集群的 API 服务器上，直到 用户或者控制器进程显式地 将其删除
    - 控制层组件会在 Pod 个数超出所配置的阈值 （根据 kube-controller-manager 的 terminated-pod-gc-threshold 设置）时 删除已终止的 Pod

#### Init 容器

##### 理解 Init 容器

- Pod 也可以有一个或多个先于应用容器启动的 Init 容器
- Init 容器与普通容器的不同点
  - 它们总是运行到完成。
  - 每个都必须在下一个启动之前成功完成。
  - Init 容器不支持 lifecycle、livenessProbe、readinessProbe 和 startupProbe
- 如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止；如果 Pod 对应的 restartPolicy 值为 "Never"，Kubernetes 不会重新启动 Pod
- 为 Pod 设置 Init 容器需要在 Pod 的 spec 中添加 initContainers 字段
- 如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。 每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时， Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行

##### 使用 Init 容器

- 下面的例子定义了一个具有 2 个 Init 容器的简单 Pod
  - 第一个等待 myservice 启动， 第二个等待 mydb 启动
  - 一旦这两个 Init 容器 都启动完成，Pod 将启动 spec 节中的应用容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: ["sh", "-c", "echo The app is running! && sleep 3600"]
  initContainers:
    - name: init-myservice
      image: busybox:1.28
      command:
        [
          "sh",
          "-c",
          "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done",
        ]
    - name: init-mydb
      image: busybox:1.28
      command:
        [
          "sh",
          "-c",
          "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done",
        ]
```

- 如下为创建这些 Service 的配置文件

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

- 查看 Pod 内 Init 容器的日志

  - kubectl logs myapp-pod -c init-myservice # 查看第一个 Init 容器
  - kubectl logs myapp-pod -c init-mydb # 查看第二个 Init 容器

- 具体行为
  - 每个 Init 容器会在网络和数据卷初始化之后按顺序启动；每个 Init 容器成功退出后才会启动下一个 Init 容器
  - 如果 Pod 重启，所有 Init 容器必须重新执行
  - 因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的代码应该是幂等的
  - 资源
    - 所有 Init 容器上定义的任何特定资源的 limit 或 request 的最大值，作为 Pod 有效初始 request/limit
    - Pod 对资源的 有效 limit/request 是如下两者的较大者：
      - 所有应用容器对某个资源的 limit/request 之和
      - 对某个资源的有效初始 limit/request
    - 基于有效 limit/request 完成调度，这意味着 Init 容器能够为初始化过程预留资源， 这些资源在 Pod 生命周期过程中并没有被使用。
    - Pod 的 有效 QoS 层 ，与 Init 容器和应用容器的一样。
  - Pod 重启的原因
    - 用户更新 Pod 的规约导致 Init 容器镜像发生改变。Init 容器镜像的变更会引起 Pod 重启。 应用容器镜像的变更仅会重启应用容器。
    - Pod 的基础设施容器 (译者注：如 pause 容器) 被重启。这种情况不多见， 必须由具备 root 权限访问节点的人员来完成。
    - 当 restartPolicy 设置为 "Always"，Pod 中所有容器会终止而强制重启。 由于垃圾收集机制的原因，Init 容器的完成记录将会丢失。

#### Pod 拓扑分布约束

- 通过拓扑分布约束来控制 Pods 在集群内故障域 之间的分布，例如区域（Region）、可用区（Zone）、节点和其他用户自定义拓扑域
- 依赖于节点标签来标识每个节点所在的拓扑域

##### Pod 的分布约束

- 字段

  - maxSkew：描述 Pod 分布不均的程度。这是给定拓扑类型中任意两个拓扑域中 匹配的 pod 之间的最大允许差值，必须大于零
    - 当 whenUnsatisfiable 等于 "DoNotSchedule" 时，maxSkew 是目标拓扑域 中匹配的 Pod 数与全局最小值之间可存在的差异
    - 当 whenUnsatisfiable 等于 "ScheduleAnyway" 时，调度器会更为偏向能够降低偏差值的拓扑域
  - topologyKey 是节点标签的键。如果两个节点使用此键标记并且具有相同的标签值， 则调度器会将这两个节点视为处于同一拓扑域中。调度器试图在每个拓扑域中放置数量均衡的 Pod
  - whenUnsatisfiable 指示如果 Pod 不满足分布约束时如何处理：
    - DoNotSchedule（默认）告诉调度器不要调度
    - ScheduleAnyway 告诉调度器仍然继续调度，只是根据如何能将偏差最小化来对 节点进行排序
  - labelSelector 用于查找匹配的 pod。匹配此标签的 Pod 将被统计，以确定相应 拓扑域中 Pod 的数量

- 示例如下

  ```yaml
  kind: Pod
  apiVersion: v1
  metadata:
    name: mypod
    labels:
      foo: bar
  spec:
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            foo: bar
    containers:
      - name: pause
        image: k8s.gcr.io/pause:3.1
  ```

  - topologyKey: zone，意味着均匀分布将只应用于存在标签键值对为 "zone:<any value>" 的节点
  - whenUnsatisfiable: DoNotSchedule 告诉调度器如果新的 Pod 不满足约束， 则让它保持悬决状态

- 多个 TopologySpreadConstraints 示例

  ```yaml
  kind: Pod
  apiVersion: v1
  metadata:
    name: mypod
    labels:
      foo: bar
  spec:
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            foo: bar
      - maxSkew: 1
        topologyKey: node
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            foo: bar
    containers:
      - name: pause
        image: k8s.gcr.io/pause:3.1
  ```

  - 多个约束之间可能存在冲突

- 约定

  - 只有与新的 Pod 具有相同命名空间的 Pod 才能作为匹配候选者
  - 没有 topologySpreadConstraints[*].topologyKey 的节点将被忽略
    - 位于这些节点上的 Pod 不影响 maxSkew 的计算
    - 新的 Pod 没有机会被调度到这类节点上
  - 如果新 Pod 的 topologySpreadConstraints[*].labelSelector 与自身的标签不匹配，在调度之后，集群的不平衡程度保持不变
  - 如果新 Pod 定义了 spec.nodeSelector 或 spec.affinity.nodeAffinity，则不匹配的节点会被忽略

- 集群级别的默认约束

  - 为集群设置默认的拓扑分布约束也是可能的。默认拓扑分布约束在且仅在以下条件满足 时才会应用到 Pod 上
    - Pod 没有在其 .spec.topologySpreadConstraints 设置任何约束
    - Pod 隶属于某个服务、副本控制器、ReplicaSet 或 StatefulSet
  - 示例

  ```yaml
  apiVersion: kubescheduler.config.k8s.io/v1beta1
  kind: KubeSchedulerConfiguration

  profiles:
    - pluginConfig:
        - name: PodTopologySpread
          args:
            defaultConstraints:
              - maxSkew: 1
                topologyKey: topology.kubernetes.io/zone
                whenUnsatisfiable: ScheduleAnyway
  ```

  - 内部默认约束

    - 当启用了 DefaultPodTopologySpread 特性门控时，原来的 SelectorSpread 插件会被禁用，kube-scheduler 会使用下面的默认拓扑约束作为 PodTopologySpread 插件的配置

    ```yaml
    defaultConstraints:
      - maxSkew: 3
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: ScheduleAnyway
      - maxSkew: 5
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
    ```

- 与 PodAffinity/PodAntiAffinity 相比较
  - 与“亲和性”相关的指令控制 Pod 的调度方式
    - 对于 PodAffinity，你可以尝试将任意数量的 Pod 集中到符合条件的拓扑域中
    - 对于 PodAntiAffinity，只能将一个 Pod 调度到某个拓扑域中

#### 干扰

- 自愿干扰和非自愿干扰

  - 非自愿干扰：Pod 不会消失，除非有人（用户或控制器）将其销毁，或者出现了不可避免的硬件或软件系统错误
    - 节点下层物理机的硬件故障
    - 集群管理员错误地删除虚拟机（实例）
    - 云提供商或虚拟机管理程序中的故障导致的虚拟机消失
    - 内核错误
    - 节点由于集群网络隔离从集群中消失
    - 由于节点资源不足导致 pod 被驱逐
  - 自愿干扰
    - 删除 Deployment 或其他管理 Pod 的控制器
    - 更新了 Deployment 的 Pod 模板导致 Pod 重启
    - 直接删除 Pod（例如，因为误操作）
    - 排空（drain）节点进行修复或升级。
    - 从集群中排空节点以缩小集群（了解集群自动扩缩）
    - 从节点中移除一个 Pod，以允许其他 Pod 使用该节点

- 处理干扰

  - 确保 Pod 在请求中给出所需资源
  - 如果需要更高的可用性，请复制应用程序。 （了解有关运行多副本的无状态 和有状态应用程序的信息）
  - 为了在运行复制应用程序时获得更高的可用性，请跨机架（使用 反亲和性）或跨区域 （如果使用多区域集群）扩展应用程序

- 干扰预算
  - 应用程序所有者可以为每个应用程序创建 PodDisruptionBudget 对象（PDB），PDB 将限制在同一时间因自愿干扰导致的复制应用程序中宕机的 pod 数量
    - 例如，基于票选机制的应用程序希望确保运行的副本数永远不会低于仲裁所需的数量
  - 集群管理员和托管提供商应该使用遵循 Pod Disruption Budgets 的接口 （通过调用 Eviction API）， 而不是直接删除 Pod 或 Deployment
    - 例如，kubectl drain 命令可以用来标记某个节点即将停止服务。 运行 kubectl drain 命令时，工具会尝试驱逐机器上的所有 Pod
  - PDB 指定应用程序可以容忍的副本数量（相当于应该有多少副本）
    - 例如，具有 .spec.replicas: 5 的 Deployment 在任何时间都应该有 5 个 Pod。 如果 PDB 允许其在某一时刻有 4 个副本，那么驱逐 API 将允许同一时刻仅有一个而不是两个 Pod 自愿干扰
  - 使用标签选择器来指定构成应用程序的一组 Pod，这与应用程序的控制器（Deployment，StatefulSet 等） 选择 Pod 的逻辑一样；Pod 控制器的 .spec.replicas 计算“预期的” Pod 数量。 根据 Pod 对象的 .metadata.ownerReferences 字段来发现控制器
  - PDB 不能阻止非自愿干扰的发生，但是确实会计入 预算

#### 临时容器

- 由于 Pod 是一次性且可替换的，因此一旦 Pod 创建，就无法将容器加入到 Pod 中
- 有时有必要检查现有 Pod 的状态。例如，对于难以复现的故障进行排查。 在这些场景中，可以在现有 Pod 中运行临时容器来检查其状态并运行任意命令
- 临时容器
  - 临时容器缺少对资源或执行的保证，并且永远不会自动重启， 因此不适用于构建应用程序，同样使用 ContainerSpec 描述，但许多字段是不兼容和不允许的
    - 临时容器没有端口配置，因此像 ports，livenessProbe，readinessProbe 这样的字段是不允许的
    - Pod 资源分配是不可变的，因此 resources 配置是不允许的
    - 更多见 [EphemeralContainer 参考文档](http://localhost:1313/docs/reference/generated/kubernetes-api/v1.20/#ephemeralcontainer-v1-core)
  - 临时容器是使用 API 中的一种特殊的 ephemeralcontainers 处理器进行创建的， 而不是直接添加到 pod.spec 段，因此无法使用 kubectl edit 来添加一个临时容器
  - 与常规容器一样，将临时容器添加到 Pod 后，将不能更改或删除临时容器
- 用途

  - 当由于容器崩溃或容器镜像不包含调试工具而导致 kubectl exec 无用时， 临时容器对于交互式故障排查很有用
  - 示例

  ```json
  {
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
      "name": "example-pod"
    },
    "ephemeralContainers": [
      {
        "command": ["sh"],
        "image": "busybox",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "terminationMessagePolicy": "File"
      }
    ]
  }
  ```

  - 使用如下命令更新已运行的临时容器 example-pod

  ```sh
  kubectl replace --raw /api/v1/namespaces/default/pods/example-pod/ephemeralcontainers  -f ec.json
  ```

  - 使用以下命令连接到新的临时容器

  ```sh
  kubectl attach -it example-pod -c debugger
  ```
