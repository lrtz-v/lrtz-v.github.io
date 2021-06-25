---
title: Kubernetes工作资源
date: 2020-12-26 20:30:01
tags: [k8s, Kubernetes, kubernetes]
description: Kubernetes工作资源介绍
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
            command: ["/bin/sh", "-c", "echo Hello Test Start> /tmp/mylog"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "echo Hello Test End> /tmp/app/mylog"]
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

  - 方式 1

    ```bash
    kubectl --record deployment.v1.apps/app-deployment set image deployment.v1.apps/app-deployment app=app:1.0.1
    ```

  - 方式 2

    ```bash
    kubectl set image deployment/app-deployment app=app:1.0.2 --record
    ```

  - 方式 3

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

  - .spec.replicas 是指定所需 Pod 的可选字段。它的默认值是 1

- 选择算符

  - .spec.selector 是指定本 Deployment 的 Pod 标签选择算符的必需字段
    - 必须匹配 .spec.template.metadata.labels，否则请求会被 API 拒绝

- 策略

  - .spec.strategy 策略指定用于用新 Pods 替换旧 Pods 的策略
  - .spec.strategy.type 可以是 “Recreate” 或 “RollingUpdate”(默认)

  - 重新创建 Deployment

    - 如果 .spec.strategy.type==Recreate，在创建新 Pods 之前，所有现有的 Pods 会被杀死

  - 滚动更新 Deployment
    - Deployment 会在 .spec.strategy.type==RollingUpdate 时，采取滚动更新的方式更新 Pods，可以指定 maxUnavailable 和 maxSurge 来控制滚动更新过程
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

- DaemonSet 确保所有符合条件的节点都运行该 Pod 的一个副本
- DaemonSet Pods 由 DaemonSet 控制器创建和调度，带来了以下问题
  - Pod 行为的不一致性：正常 Pod 在被创建后等待调度时处于 Pending 状态， DaemonSet Pods 创建后不会处于 Pending 状态下。这使用户感到困惑
  - Pod 抢占 由默认调度器处理。启用抢占后，DaemonSet 控制器将在不考虑 Pod 优先级和抢占 的情况下制定调度决策
- ScheduleDaemonSetPods 允许您使用默认调度器而不是 DaemonSet 控制器来调度 DaemonSets，方法是将 NodeAffinity 条件而不是 .spec.nodeName 条件添加到 DaemonSet Pods
- 默认调度器接下来将 Pod 绑定到目标主机
  - 如果 DaemonSet Pod 的节点亲和性配置已存在，则被替换
  - DaemonSet 控制器仅在创建或修改 DaemonSet Pod 时执行这些操作， 并且不会更改 DaemonSet 的 spec.template
- 系统会自动添加 node.kubernetes.io/unschedulable：NoSchedule 容忍度到 DaemonSet Pods。在调度 DaemonSet Pod 时，默认调度器会忽略 unschedulable 节点

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchFields:
          - key: metadata.name
            operator: In
            values:
              - target-host-name
```

### 与 Daemon Pods 通信

- 推送（Push）：配置 DaemonSet 中的 Pod，将更新发送到另一个服务，例如统计数据库。 这些服务没有客户端
- NodeIP 和已知端口：DaemonSet 中的 Pod 可以使用 hostPort，从而可以通过节点 IP 访问到 Pod。客户端能通过某种方法获取节点 IP 列表，并且基于此也可以获取到相应的端口
- DNS：创建具有相同 Pod 选择算符的 无头服务， 通过使用 endpoints 资源或从 DNS 中检索到多个 A 记录来发现 DaemonSet
- Service：创建具有相同 Pod 选择算符的服务，并使用该服务随机访问到某个节点上的 守护进程（没有办法访问到特定节点）

### 更新 DaemonSet

- 如果节点的标签被修改，DaemonSet 将立刻向新匹配上的节点添加 Pod， 同时删除不匹配的节点上的 Pod
- 你可以修改 DaemonSet 创建的 Pod。不过并非 Pod 的所有字段都可更新
- 可以删除一个 DaemonSet

## Job

- Job 会创建一个或者多个 Pods，并确保指定数量的 Pods 成功终止
- 随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pods

### Job 示例

- 计算 π 到小数点后 100 位，并将结果打印出来

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: resouer/ubuntu-bc
          imagePullPolicy: Never
          command: ["sh", "-c", "echo 'scale=100; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
  activeDeadlineSeconds: 100
```

- kubectl logs $pods

### 编写 Job 规约

- Job 的并行执行

  - 非并行 Job
    - 通常只启动一个 Pod，除非该 Pod 失败
    - 当 Pod 成功终止时，立即视 Job 为完成状态
    - 可以不设置 spec.completions 和 spec.parallelism。 这两个属性都不设置时，均取默认值 1
  - 具有 确定完成计数 的并行 Job
    - .spec.completions 为所需要的完成个数，若不设置 spec.completions，默认值为 .spec.parallelism，其默认值为 1
    - Job 用来代表整个任务，当对应于 1 和 .spec.completions 之间的每个整数都存在 一个成功的 Pod 时，Job 被视为完成
    - 尚未实现：每个 Pod 收到一个介于 1 和 spec.completions 之间的不同索引值
  - 带 工作队列 的并行 Job
    - 不可以设置 .spec.completions，但要将.spec.parallelism 设置为一个非负整数
    - 多个 Pod 之间必须相互协调，或者借助外部服务确定每个 Pod 要处理哪个工作条目。 例如，任一 Pod 都可以从工作队列中取走最多 N 个工作条目
    - 每个 Pod 都可以独立确定是否其它 Pod 都已完成，进而确定 Job 是否完成
    - 当 Job 中 任何 Pod 成功终止，不再创建新 Pod
    - 一旦至少 1 个 Pod 成功完成，并且所有 Pod 都已终止，即可宣告 Job 成功完成
    - 一旦任何 Pod 成功退出，任何其它 Pod 都不应再对此任务执行任何操作或生成任何输出。 所有 Pod 都应启动退出过程

- 控制并行性
  - 并行性请求（.spec.parallelism）可以设置为任何非负整数。 如果未设置，则默认为 1。 如果设置为 0，则 Job 相当于启动之后便被暂停，直到此值被增加
  - 实际并行性（在任意时刻运行状态的 Pods 个数）可能比并行性请求略大或略小， 原因如下
    - 对于 确定完成计数 Job，实际上并行执行的 Pods 个数不会超出剩余的完成数。 如果 .spec.parallelism 值较高，会被忽略。
    - 对于 工作队列 Job，有任何 Job 成功结束之后，不会有新的 Pod 启动。 不过，剩下的 Pods 允许执行完毕。
    - 如果 Job 控制器 没有来得及作出响应，或者
    - 如果 Job 控制器因为任何原因（例如，缺少 ResourceQuota 或者没有权限）无法创建 Pods。 Pods 个数可能比请求的 - 数目小。
    - Job 控制器可能会因为之前同一 Job 中 Pod 失效次数过多而压制新 Pod 的创建。
    - 当 Pod 处于体面终止进程中，需要一定时间才能停止。

### 处理 Pod 和容器失效

- Pod 中的容器可能因为多种不同原因失效

  - .spec.template.spec.restartPolicy = "OnFailure"
    - Pod 则继续留在当前节点，但容器会被重新运行
  - .spec.template.spec.restartPolicy = "Never"
    - Job 控制器会启动一个新的 Pod

- 即使将 .spec.parallelism 设置为 1，且将 .spec.completions 设置为 1，并且 .spec.template.spec.restartPolicy 设置为 "Never"，同一程序仍然有可能被启动两次

- Pod 回退失效策略
  - 在有些情形下，我们可能希望 Job 在经历若干次重试之后直接进入失败状态，因为这很可能意味着遇到了配置错误
  - 可以将 .spec.backoffLimit 设置为视 Job 为失败之前的重试次数，失效回退的限制值默认为 6，一旦重试次数到达 .spec.backoffLimit 所设的上限，Job 会被标记为失败， 其中运行的 Pods 都会被终止
  - 与 Job 相关的失效的 Pod 会被 Job 控制器重建，并且以指数型回退计算重试延迟 （从 10 秒、20 秒到 40 秒，最多 6 分钟）。 当 Job 的 Pod 被删除时，或者 Pod 成功时没有其它 Pod 处于失败状态，失效回退的次数也会被重置（为 0）

### Job 终止与清理

- Job 完成时不会再创建新的 Pod，不过已有的 Pod 也不会被删除。 保留这些 Pod 使得你可以查看已完成的 Pod 的日志输出，以便检查错误、警告 或者其它诊断性输出

- 你可以为 Job 的 .spec.activeDeadlineSeconds 设置一个秒数值，值适用于 Job 的整个生命期，无论 Job 创建了多少个 Pod。 一旦 Job 运行时间达到 activeDeadlineSeconds 秒，其所有运行中的 Pod 都会被终止，并且 Job 的状态更新为 type: Failed 及 reason: DeadlineExceeded
  - Job 的 .spec.activeDeadlineSeconds 优先级高于其 .spec.backoffLimit 设置

### 自动清理完成的 Job

- 如果 Job 由某种更高级别的控制器来管理，例如 CronJobs， 则 Job 可以被 CronJob 基于特定的根据容量裁定的清理策略清理掉

- 使用由 TTL 控制器所提供 的 TTL 机制自动清理 Job

  - 设置 Job 的 .spec.ttlSecondsAfterFinished 字段，可以让该控制器清理掉 已结束的资源
  - 如果该字段设置为 0，Job 在结束之后立即成为可被自动删除的对象

  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: pi
  spec:
    template:
      spec:
        containers:
          - name: pi
            image: resouer/ubuntu-bc
            imagePullPolicy: Never
            command: ["sh", "-c", "echo 'scale=100; 4*a(1)' | bc -l "]
        restartPolicy: Never
    backoffLimit: 4
    activeDeadlineSeconds: 100
    ttlSecondsAfterFinished: 100
  ```

## CronJob

- Cron Job 创建基于时间调度的 Jobs，对于创建周期性的、反复重复的任务很有用，例如执行数据备份或者发送邮件。 CronJobs 也可以用来计划在指定时间来执行的独立任务，例如计划当集群看起来很空闲时 执行某个 Job

### CronJob 示例

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

### CronJob 限制

- CronJob 根据其计划编排，在每次该执行任务的时候大约会创建一个 Job
  - 在某些情况下，可能会创建两个 Job，或者不会创建任何 Job
  - 如果 startingDeadlineSeconds 设置为很大的数值或未设置（默认），并且 concurrencyPolicy 设置为 Allow，则作业将始终至少运行一次；如果 startingDeadlineSeconds 是 200，则控制器会统计在过去 200 秒中错过了多少次 Job
  - CronJob 控制器（Controller） 检查从上一次调度的时间点到现在所错过了调度次数。如果错过的调度次数超过 100 次， 那么它就不会启动这个任务，并记录这个错误
  - 未能在调度时间内创建 CronJob，则计为错过

### CronJonV2

- --feature-gates="CronJobControllerV2=true"

## 资源回收

### 宿主与附属

- 每个附属对象具有一个指向其所属对象的 metadata.ownerReferences 字段

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-repset
spec:
  replicas: 3
  selector:
    matchLabels:
      pod-is-for: garbage-collection-example
  template:
    metadata:
      labels:
        pod-is-for: garbage-collection-example
    spec:
      containers:
        - name: app
          image: app:1.0.0
          imagePullPolicy: Never
```

- 通过 kubectl get pods --output=yaml 查看 ownerReferences

  - 显示了 Pod 的属主是名为 my-repset 的 ReplicaSet

    ```yaml
    ownerReferences:
      - apiVersion: apps/v1
        blockOwnerDeletion: true
        controller: true
        kind: ReplicaSet
        name: my-repset
        uid: 00c1a24c-1418-4c3b-be9d-326915144427
    ```

### 控制垃圾收集器删除附属

- 当你删除对象时，可以指定该对象的附属是否也自动删除

  - 不自动删除它的附属，这些附属被称作 孤立对象（Orphaned）
  - 自动删除附属的行为也称为 级联删除（Cascading Deletion）
    - 级联删除模式
      - 后台（Background） 模式
      - 前台（Foreground） 模式

- 前台级联删除
  - 根对象首先进入 deletion in progress 状态。 在 deletion in progress 状态，会有如下的情况
    - 对象仍然可以通过 REST API 可见
    - 对象的 deletionTimestamp 字段被设置
    - 对象的 metadata.finalizers 字段包含值 foregroundDeletion
  - 一旦对象被设置为 deletion in progress 状态，垃圾收集器会删除对象的所有附属
  - 垃圾收集器在删除了所有有阻塞能力的附属（对象的 ownerReference.blockOwnerDeletion=true） 之后，删除属主对象
  - 只有设置了 ownerReference.blockOwnerDeletion 值的附属才能阻止删除属主对象
- 后台级联删除

  - Kubernetes 会立即删除属主对象，之后垃圾收集器 会在后台删除其附属对象

- 设置级联删除策略

  - 通过为属主对象设置 deleteOptions.propagationPolicy 字段，可以控制级联删除策略
  - 可能的取值包括：Orphan、Foreground 或者 Background

- 示例

```bash
下面是一个在后台删除附属对象的示例：

kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
  -H "Content-Type: application/json"

下面是一个在前台中删除附属对象的示例：

kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
  -H "Content-Type: application/json"

下面是一个令附属成为孤立对象的示例：

kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
  -H "Content-Type: application/json"
```

- kubectl 命令也支持级联删除
  - 通过设置 --cascade 为 true，可以使用 kubectl 自动删除附属对象
  - 设置 --cascade 为 false，会使附属对象成为孤立附属对象。 --cascade 的默认值是 true
