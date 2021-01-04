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
  - 出现场景
    - 配额（Quota）不足
    - 就绪探测（Readiness Probe）失败
    - 镜像拉取错误
    - 权限不足
    - 限制范围（Limit Ranges）问题
    - 应用程序运行时的配置错误
  - 检测方式
