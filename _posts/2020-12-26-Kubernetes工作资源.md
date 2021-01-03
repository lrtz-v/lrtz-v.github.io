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


