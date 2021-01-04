---
title: Kubernetes工作资源
date: 2020-12-26 20:30:01
tags: [k8s, Kubernetes, kubernetes]
---

# Kubernetes 工作资源

## 第一个自定义服务

- 为了学习 Kubernetes 工作资源使用姿势，我们自己定义一个 Http 服务，代码如下

```golang
package main

import (
  "fmt"
  "net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "<h1>Test Homepage</h1>")
}

func main() {
  http.HandleFunc("/", indexHandler)
  http.ListenAndServe(":8080", nil)
}

```

- Dockerfile 如下

```Dockerfile
FROM golang:latest AS build

WORKDIR /app

ENV CGO_ENABLED=0
ENV GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
ENV GO111MODULE=on
ENV GOOS=linux

COPY . .
RUN go mod tidy
RUN go build -o app .


FROM alpine:latest
WORKDIR /root/
COPY --from=build /app .
CMD ["./app"]
```

- [完整代码地址](https://github.com/lrtz-v/Tutorial/tree/master/k8s/app)

## Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    - name: app
spec:
  # hostAliases:
  #   - ip: "10.1.2.3"
  #     hostnames:
  #       - "foo.remote"
  #       - "bar.remote"
  shareProcessNamespace: true
  containers:
    - name: app
      image: app:1.0.0
      imagePullPolicy: Never
      lifecycle:
        postStart:
          exec:
            command:
              [
                "/bin/sh",
                "-c",
                "echo Hello Test Start> /tmp/mylog",
              ]
        preStop:
          exec:
            command:
              [
                "/bin/sh",
                "-c",
                "echo Hello Test End> /tmp/app/mylog",
              ]
      volumeMounts:
        - mountPath: /tmp/app
          name: app-volume
      ports:
        - containerPort: 8080
          hostPort: 8080
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/app/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
  volumes:
    - name: app-volume
      emptyDir: {}
```

## ReplicaSet

- RepicaSet 是通过一组字段来定义的，包括一个用来识别可获得的 Pod 的集合的选择算符、一个用来标明应该维护的副本个数的数值、一个用来指定应该创建新 Pod 以满足副本个数条件时要使用的 Pod 模板等等。 每个 ReplicaSet 都通过根据需要创建和 删除 Pod 以使得副本个数达到期望值， 进而实现其存在价值。当 ReplicaSet 需要创建新的 Pod 时，会使用所提供的 Pod 模板。
- ReplicaSet 通过 Pod 上的 metadata.ownerReferences 字段连接到附属 Pod，该字段给出当前对象的属主资源。 ReplicaSet 所获得的 Pod 都在其 ownerReferences 字段中包含了属主 ReplicaSet 的标识信息。正是通过这一连接，ReplicaSet 知道它所维护的 Pod 集合的状态， 并据此计划其操作行为
- ReplicaSet 使用其选择算符来辨识要获得的 Pod 集合

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
  labels:
    app: app-rs
    tier: app-rs
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: app-rs
  template:
    metadata:
      labels:
        tier: app-rs
    spec:
      containers:
      - name: app
        image: app:1.0.0
        imagePullPolicy: Never
```

- ReplicaSet 可以被 HPA 自动缩放

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

## Deployments

- Deployment 控制器为 Pods 和 ReplicaSets 提供声明式的更新能力
  - Deployment 控制器每次注意到新的 Deployment 时，都会创建一个 ReplicaSet 以启动所需的 Pods
  - 如果更新了 Deployment，则控制标签匹配 .spec.selector 但模板不匹配 .spec.template 的 Pods 的现有 ReplicaSet 被缩容。最终，新的 ReplicaSet 缩放为 .spec.replicas 个副本， 所有旧 ReplicaSets 缩放为 0 个副本

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: app:1.0.0
        ports:
        - containerPort: 8080
```

```bash
➜  ~: k get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           7s

➜  ~: k get ReplicaSet
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-77d4987499   3         3         3       10s

➜  ~: k get pod
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-77d4987499-5jkj2   1/1     Running   0          23s
app-deployment-77d4987499-g7wcn   1/1     Running   0          23s
app-deployment-77d4987499-h2qsm   1/1     Running   0          23s
```

### 更新 Deplyment

- 更新镜像
  - 方式1

    ```bash
    kubectl --record deployment.v1.apps/app-deployment set image deployment.v1.apps/app-deployment app=app:1.0.1
    ```

  - 方式2

    ```bash
    kubectl set image deployment/app-deployment app=app:1.0.2 --record
    ```

  - 方式3

    ```bash
    kubectl edit deployment.v1.apps/app-deployment
    ```

- deployment 事件记录如下

  ```bash
  NewReplicaSet:   app-deployment-76fb7489cd (3/3 replicas created)
  Events:
    Type    Reason             Age                 From                   Message
    ----    ------             ----                ----                   -------
    Normal  ScalingReplicaSet  17m                 deployment-controller  Scaled up replica set app-deployment-77d4987499 to 3
    Normal  ScalingReplicaSet  4m30s               deployment-controller  Scaled up replica set app-deployment-64fcd57fb to 1
    Normal  ScalingReplicaSet  4m28s               deployment-controller  Scaled down replica set app-deployment-77d4987499 to 2
    Normal  ScalingReplicaSet  4m28s               deployment-controller  Scaled up replica set app-deployment-64fcd57fb to 2
    Normal  ScalingReplicaSet  4m27s               deployment-controller  Scaled down replica set app-deployment-77d4987499 to 1
    Normal  ScalingReplicaSet  4m27s               deployment-controller  Scaled up replica set app-deployment-64fcd57fb to 3
    Normal  ScalingReplicaSet  4m26s               deployment-controller  Scaled down replica set app-deployment-77d4987499 to 0
    Normal  ScalingReplicaSet  107s                deployment-controller  Scaled up replica set app-deployment-647bb69489 to 1
    Normal  ScalingReplicaSet  106s                deployment-controller  Scaled down replica set app-deployment-64fcd57fb to 2
    Normal  ScalingReplicaSet  106s                deployment-controller  Scaled up replica set app-deployment-647bb69489 to 2
  ```

  - 当第一次创建 Deployment 时，它创建了一个 ReplicaSet 并将其直接扩容至 3 个副本
  - 更新 Deployment 时，它创建了一个新的 ReplicaSet ，并将其扩容为 1，然后将旧 ReplicaSet 缩容到 2， 以便至少有 2 个 Pod 可用且最多创建 4 个 Pod
  - 然后，它使用相同的滚动更新策略继续对新的 ReplicaSet 扩容并对旧的 ReplicaSet 缩容。 最后，你将有 3 个可用的副本在新的 ReplicaSet 中，旧 ReplicaSet 将缩容到 0

### 回滚 Deployment

- 检查上线版本
  
  ```bash
  ➜  ~: kubectl rollout history deployment.v1.apps/app-deployment
  deployment.apps/app-deployment
  REVISION  CHANGE-CAUSE
  1         <none>
  2         kubectl deployment.v1.apps/app-deployment set image deployment.v1.apps/app-deployment app=app:1.0.1 --record=true
  3         kubectl set image deployment/app-deployment app=app:1.0.2 --record=true
  4         kubectl set image deployment/app-deployment app=app:1.0.2 --record=true
  ```

- 查看修订历史的详细信息

  ```bash

  ➜  ~: kubectl rollout history deployment.v1.apps/app-deployment --revision=2
  deployment.apps/app-deployment with revision #2
  Pod Template:
    Labels: app=app
    pod-template-hash=64fcd57fb
    Annotations: kubernetes.io/change-cause:
      kubectl deployment.v1.apps/app-deployment set image deployment.v1.apps/app-deployment app=app:1.0.1 --record=true
    Containers:
    app:
      Image: app:1.0.1
      Port: 8080/TCP
      Host Port: 0/TCP
      Environment: <none>
      Mounts: <none>
    Volumes: <none>
  ```

- 回滚到之前的修订版本

  - 撤消当前上线并回滚到以前的修订版本

  ```bash
  ➜  ~: kubectl rollout undo deployment.v1.apps/app-deployment
  deployment.apps/app-deployment rolled back
  ```

  - 会滚到指定版本
  
  ```bash
  ➜  ~: kubectl rollout undo deployment.v1.apps/app-deployment --to-revision=2
  deployment.apps/app-deployment rolled back
  ```

### 缩放 Deployment

- 缩放
  - 命令

    ```bash
    ➜  ~: kubectl scale deployment.v1.apps/app-deployment --replicas=3
    ```

  - 水平自动缩放

    ```bash
    ➜  ~: kubectl autoscale deployment.v1.apps/app-deployment --min=10 --max=15 --cpu-percent=80
    ```

- 比例缩放

  - RollingUpdate 的 Deployment 支持同时运行应用程序的多个版本。当自动缩放器缩放处于上线进程（仍在进行中或暂停）中的 RollingUpdate Deployment 时， Deployment 控制器会平衡现有的活跃状态的 ReplicaSets（含 Pods 的 ReplicaSets）中的额外副本， 以降低风险。这称为 比例缩放（Proportional Scaling）

### 暂停、恢复 Deployment

- 在触发一个或多个更新之前暂停 Deployment，然后再恢复其执行

- 例如

  ```bash
  ➜  ~: kubectl rollout pause deployment.v1.apps/app-deployment
  deployment.apps/app-deployment paused
  
  ➜  ~: kubectl set resources deployment.v1.apps/app-deployment -c=app --limits=cpu=200m,memory=512Mi
  deployment.apps/app-deployment resource requirements updated

  ➜  ~: kubectl rollout resume deployment.v1.apps/app-deployment
  deployment.apps/nginx-deployment resumed

  ➜  ~: kubectl get rs -w
  deployment.apps/nginx-deployment resumed
  ```

### Deployment 状态

- 状态查询
  
  ```bash
  ➜  ~: kubectl rollout status deployment.v1.apps/nginx-deployment
  ```

- 进行中的 Deployment
  - 出现场景
    - Deployment 创建新的 ReplicaSet
    - Deployment 正在为其最新的 ReplicaSet 扩容
    - Deployment 正在为其旧有的 ReplicaSet(s) 缩容
    - 新的 Pods 已经就绪或者可用（就绪至少持续了 MinReadySeconds 秒）

- 完成的 Deployment
  - 出现场景
    - 与 Deployment 关联的所有副本都已更新到指定的最新版本，这意味着之前请求的所有更新都已完成
    - 与 Deployment 关联的所有副本都可用
    - 未运行 Deployment 的旧副本

- 失败的 Deployment
  - Deployment 可能一直处于未完成状态，出现场景如下
    - 配额（Quota）不足
    - 就绪探测（Readiness Probe）失败
    - 镜像拉取错误
    - 权限不足
    - 限制范围（Limit Ranges）问题
    - 应用程序运行时的配置错误
  - 检测方式
    - 在 Deployment 规约中指定截止时间参数：.spec.progressDeadlineSeconds

    ```bash
    ➜  ~: kubectl patch deployment.v1.apps/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
    ```

    - 一旦超过 Deployment 进度限期，Kubernetes 将更新状态和进度状况的原因

    ```bash
    Conditions:
      Type            Status  Reason
      ----            ------  ------
      Available       True    MinimumReplicasAvailable
      Progressing     False   ProgressDeadlineExceeded
      ReplicaFailure  True    FailedCreate
    ```

### 清理策略

- 在 Deployment 中设置 .spec.revisionHistoryLimit 字段以指定保留此 Deployment 的多少个旧有 ReplicaSet。其余的 ReplicaSet 将在后台被垃圾回收。 默认情况下，此值为 10
  - 将此字段设置为 0 将导致 Deployment 的所有历史记录被清空，因此 Deployment 将无法回滚

### Deployment 规约

- apiVersion，kind 和 metadata

- Pod 模板
  - .spec 中只有 .spec.template 和 .spec.selector 是必需的字段
    - .spec.template 是一个 Pod 模板

- 副本
  - .spec.replicas 是指定所需 Pod 的可选字段。它的默认值是1

- 选择算符
  - .spec.selector 是指定本 Deployment 的 Pod 标签选择算符的必需字段
    - 必须匹配 .spec.template.metadata.labels，否则请求会被 API 拒绝

- 策略
  - .spec.strategy 策略指定用于用新 Pods 替换旧 Pods 的策略
  - .spec.strategy.type 可以是 “Recreate” 或 “RollingUpdate”(默认)

  - 重新创建 Deployment
    - 如果 .spec.strategy.type==Recreate，在创建新 Pods 之前，所有现有的 Pods 会被杀死

  - 滚动更新 Deployment
    - Deployment 会在 .spec.strategy.type==RollingUpdate时，采取滚动更新的方式更新 Pods，可以指定 maxUnavailable 和 maxSurge 来控制滚动更新过程
      - maxUnavailable
        - .spec.strategy.rollingUpdate.maxUnavailable 是一个可选字段，用来指定 更新过程中不可用的 Pod 的个数上限；该值可以是绝对数字（例如，5），也可以是 所需 Pods 的百分比（例如，10%）。百分比值会转换成绝对数并去除小数部分。 如果 .spec.strategy.rollingUpdate.maxSurge 为 0，则此值不能为 0。 默认值为 25%
      - maxSurge
        - .spec.strategy.rollingUpdate.maxSurge 是一个可选字段，用来指定可以创建的超出 期望 Pod 个数的 Pod 数量。此值可以是绝对数（例如，5）或所需 Pods 的百分比（例如，10%）。 如果 MaxUnavailable 为 0，则此值不能为 0。百分比值会通过向上取整转换为绝对数。 此字段的默认值为 25%

- 进度期限秒数
  - .spec.progressDeadlineSeconds 是一个可选字段，用于指定系统在报告 Deployment 进展失败 之前等待 Deployment 取得进展的秒数
  - 如果指定，则此字段值需要大于 .spec.minReadySeconds 取值

- 最短就绪时间
  - .spec.minReadySeconds 是一个可选字段，用于指定新创建的 Pod 在没有任意容器崩溃情况下的最小就绪时间， 只有超出这个时间 Pod 才被视为可用。默认值为 0（Pod 在准备就绪后立即将被视为可用）

- 修订历史限制
  - Deployment 的修订历史记录存储在它所控制的 ReplicaSets 中
  - .spec.revisionHistoryLimit 是一个可选字段，用来设定出于会滚目的所要保留的旧 ReplicaSet 数量。 这些旧 ReplicaSet 会消耗 etcd 中的资源，并占用 kubectl get rs 的输出
  - 每个 Deployment 修订版本的配置都存储在其 ReplicaSets 中；因此，一旦删除了旧的 ReplicaSet， 将失去回滚到 Deployment 的对应修订版本的能力。 默认情况下，系统保留 10 个旧 ReplicaSet，但其理想值取决于新 Deployment 的频率和稳定性

- paused
  - .spec.paused 是用于暂停和恢复 Deployment 的可选布尔字段。
  - 暂停的 Deployment 和未暂停的 Deployment 的唯一区别是
    - Deployment 处于暂停状态时， PodTemplateSpec 的任何修改都不会触发新的上线。 Deployment 在创建时是默认不会处于暂停状态

## StatefulSets

- StatefulSet 是用来管理有状态应用的工作负载 API 对象；可以用来管理某 Pod 集合的部署和扩缩， 并为这些 Pod 提供持久存储和持久标识符
  - 和 Deployment 类似， StatefulSet 管理基于相同容器规约的一组 Pod
  - 但和 Deployment 不同的是， StatefulSet 为它们的每个 Pod 维护了一个有粘性的 ID，这些 Pod 是基于相同的规约来创建的， 但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID

### 限制

- 给定 Pod 的存储必须由 PersistentVolume 驱动 基于所请求的 storage class 来提供，或者由管理员预先提供
- 删除或者收缩 StatefulSet 并不会删除它关联的存储卷。 这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值
- StatefulSet 当前需要 Headless Service 来负责 Pod 的网络标识。你需要负责创建此服务
- 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。 为了实现 StatefulSet 中的 Pod 可以有序地且体面地终止，可以在删除之前将 StatefulSet 缩放为 0
- 在默认 Pod 管理策略(OrderedReady) 时使用 滚动更新，可能进入需要人工干预 才能修复的损坏状态

### 示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
  labels:
    app: app
spec:
  selector:
    app: app
  clusterIP: None
  ports:
  - name: default
    protocol: TCP
    port: 8080
    targetPort: 8080
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app
spec:
  selector:
    matchLabels:
      app: app
  serviceName: "app"
  replicas: 3
  template:
    metadata:
      labels:
        app: app
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: app
        image: app:1.0.3
        imagePullPolicy: Never
        ports:
        - containerPort: 8080
          name: app
```

### Pod 选择算符

- 必须设置 StatefulSet 的 .spec.selector 字段，使之匹配其在 .spec.template.metadata.labels 中设置的标签

### Pod 标识

- StatefulSet Pod 具有唯一的标识，该标识包括顺序标识、稳定的网络标识和稳定的存储。 该标识和 Pod 是绑定的，不管它被调度在哪个节点上

- 有序索引
  - 对于具有 N 个副本的 StatefulSet，StatefulSet 中的每个 Pod 将被分配一个整数序号， 从 0 到 N-1，该序号在 StatefulSet 上是唯一的

- 稳定的网络 ID
  - StatefulSet 中的每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名
  - 组合主机名的格式为$(StatefulSet 名称)-$(序号)
  - StatefulSet 可以使用 Headless Service 控制它的 Pod 的网络域。管理域的这个服务的格式为： $(服务名称).$(命名空间).svc.cluster.local，其中 cluster.local 是集群域。 一旦每个 Pod 创建成功，就会得到一个匹配的 DNS 子域，格式为： $(pod 名称).$(所属服务的 DNS 域名)，其中所属服务由 StatefulSet 的 serviceName 域来设定。

- 稳定的存储
  - Kubernetes 为每个 VolumeClaimTemplate 创建一个 PersistentVolume
  - 在上面的示例中，每个 Pod 将会得到基于 StorageClass my-storage-class 提供的 1 Gib 的 PersistentVolume。如果没有声明 StorageClass，就会使用默认的 StorageClass。 当一个 Pod 被调度（重新调度）到节点上时，它的 volumeMounts 会挂载与其 PersistentVolumeClaims 相关联的 PersistentVolume。 请注意，当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。

- Pod 名称标签
  - 当 StatefulSet 控制器（Controller） 创建 Pod 时， 它会添加一个标签 statefulset.kubernetes.io/pod-name，该标签值设置为 Pod 名称。 这个标签允许给 StatefulSet 中的特定 Pod 绑定一个 Service

### 部署和扩缩保证

- 对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 0..N-1
- 当删除 Pod 时，它们是逆序终止的，顺序为 N-1..0
- 在将缩放操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态
- 在 Pod 终止之前，所有的继任者必须完全关闭

### 更新策略

- StatefulSet 的 .spec.updateStrategy 字段让 你可以配置和禁用掉自动滚动更新 Pod 的容器、标签、资源请求或限制、以及注解
- 当 StatefulSet 的 .spec.updateStrategy.type 设置为 OnDelete 时，它的控制器将不会自动更新 StatefulSet 中的 Pod。 用户必须手动删除 Pod 以便让控制器创建新的 Pod，以此来对 StatefulSet 的 .spec.template 的变动作出反应

- 滚动更新
  - RollingUpdate 更新策略对 StatefulSet 中的 Pod 执行自动的滚动更新。 在没有声明 .spec.updateStrategy 时，RollingUpdate 是默认配置。 当 StatefulSet 的 .spec.updateStrategy.type 被设置为 RollingUpdate 时， StatefulSet 控制器会删除和重建 StatefulSet 中的每个 Pod。 它将按照与 Pod 终止相同的顺序（从最大序号到最小序号）进行，每次更新一个 Pod。 它会等到被更新的 Pod 进入 Running 和 Ready 状态，然后再更新其前身

  - 分区
    - 通过声明 .spec.updateStrategy.rollingUpdate.partition 的方式，RollingUpdate 更新策略可以实现分区
    - 如果声明了一个分区，当 StatefulSet 的 .spec.template 被更新时， 所有序号大于等于该分区序号的 Pod 都会被更新。 所有序号小于该分区序号的 Pod 都不会被更新，并且，即使他们被删除也会依据之前的版本进行重建
    - 如果 StatefulSet 的 .spec.updateStrategy.rollingUpdate.partition 大于它的 .spec.replicas，对它的 .spec.template 的更新将不会传递到它的 Pod

  - 强制回滚
    - 在默认 Pod 管理策略(OrderedReady) 下使用 滚动更新 ，可能进入需要人工干预才能修复的损坏状态
    - 如果更新后 Pod 模板配置进入无法运行或就绪的状态（例如，由于错误的二进制文件 或应用程序级配置错误），StatefulSet 将停止回滚并等待
    - 在这种状态下，仅将 Pod 模板还原为正确的配置是不够的，StatefulSet 将继续等待损坏状态的 Pod 准备就绪（永远不会发生），然后再尝试将其恢复为正常工作配置
    - 恢复模板后，还必须删除 StatefulSet 尝试使用错误的配置来运行的 Pod。这样， StatefulSet 才会开始使用被还原的模板来重新创建 Pod

## DaemonSet

- 确保全部（或者某些）节点上运行一个 Pod 的副本。 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod

### 典型用法

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

### 编写 DaemonSet Spec

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

- Pod 模板
  - 在 DaemonSet 中的 Pod 模板必须具有一个值为 Always 的 RestartPolicy。 当该值未指定时，默认是 Always

- 仅在某些节点上运行 Pod
  - 如果指定了 .spec.template.spec.nodeSelector，DaemonSet 控制器将在能够与 Node 选择算符 匹配的节点上创建 Pod
  - 类似可以指定 .spec.template.spec.affinity，之后 DaemonSet 控制器 将在能够与节点亲和性 匹配的节点上创建 Pod

### Daemon Pods 调度
