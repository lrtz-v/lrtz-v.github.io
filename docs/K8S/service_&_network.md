---
template: main.html
tags:
  - K8S
  - Kubernetes
  - Service
---

# Service

- 将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法

## 虚拟 IP 和 Service 代理

- userspace 代理模式
- iptables 代理模式
- IPVS 代理模式

## 服务发现

- 环境变量
- DNS

## Headless Services

- 不需要或不想要负载均衡

## 发布服务（服务类型)

- Type
  - ClusterIP，通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问
  - NodePort，通过每个节点上的 IP 和静态端口（NodePort）暴露服务，通过请求 <节点 IP>:<节点 PORT>，你可以从集群的外部访问一个 NodePort 服务。NodePort 服务会路由到自动创建的 ClusterIP 服务
  - LoadBalancer，使用云提供商的负载均衡器向外部暴露服务。外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。
  - ExternalName，通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。 无需创建任何类型代理

## 服务准备

- 代码

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
	err := http.ListenAndServe(":80", nil)
	if err != nil {
		panic(err)
	}
}
```

```Dockerfile
FROM golang:latest AS build

WORKDIR /app

ENV CGO_ENABLED=0
ENV GO111MODULE=on
ENV GOOS=linux

COPY . .
RUN go mod tidy
RUN go build -o app .


FROM alpine:latest
WORKDIR /root/
COPY --from=build /app .
COPY start.sh .
CMD ["sh", "start.sh"]
```

- build

```bash
zsh ➜docker build -t app:1.0.0 .
```

## ClusterIP

- service_cluster.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      run: my-app
  replicas: 2
  template:
    metadata:
      labels:
        run: my-app
    spec:
      containers:
        - name: my-app
          image: app:1.0.0
          imagePullPolicy: Never
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    run: my-app
spec:
  type: ClusterIP
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    run: my-app
```

```bash
zsh ➜ kubectl apply -f service_cluster.yaml
```

- get resources status

```bash
zsh ➜ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
my-app-64db968db4-pldzs   1/1     Running   0          8m11s   172.17.0.5   minikube   <none>           <none>
my-app-64db968db4-xf2xj   1/1     Running   0          8m11s   172.17.0.6   minikube   <none>           <none>

zsh ➜ kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
my-app       ClusterIP   10.105.122.196   <none>        80/TCP    9m53s   run=my-app
```

- request cluster service
  - 需要从同一集群中的 Pod 中访问 ClusterIP

```bash
zsh ➜ kubectl run busybox -it --image=busybox:1.28 --restart=Never --rm
/ # wget -qO - 10.105.122.196
<h1>Test Homepage</h1>
/ # wget -qO - 172.17.0.3
<h1>Test Homepage</h1>
```

## NodePort

- service_node.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      run: my-app
  replicas: 2
  template:
    metadata:
      labels:
        run: my-app
    spec:
      containers:
        - name: my-app
          image: app:1.0.0
          imagePullPolicy: Never
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    run: my-app
spec:
  type: NodePort
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    run: my-app
```

```bash
zsh ➜ kubectl apply -f service_node.yaml
```

- get resources status

```bash
zsh ➜ kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
my-app       NodePort    10.106.216.118   <none>        80:31646/TCP   5m14s   run=my-app
```

- request

```bash
zsh ➜ NODEPORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services my-app)
zsh ➜ NODES=$(kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="InternalIP")].address }')
zsh ➜ for node in $NODES; do curl -s $node:$NODEPORT | grep -i client_address; done
```

- request if minikube

```bash
zsh ➜ minikube service my-app --url
http://127.0.0.1:50883
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

```bash
zsh ➜ curl http://127.0.0.1:50883
<h1>Test Homepage</h1>
```

## LoadBalancer

- service_loadbalabcer.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      run: my-app
  replicas: 2
  template:
    metadata:
      labels:
        run: my-app
    spec:
      containers:
        - name: my-app
          image: app:1.0.0
          imagePullPolicy: Never
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    run: my-app
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    run: my-app
```

```bash
zsh ➜ kubectl apply -f service_loadbalancer.yaml
```

- get resources status

```bash
zsh ➜ kubectl get svc -o wide
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
my-app       LoadBalancer   10.108.70.232   127.0.0.1     80:31047/TCP   4m6s   run=my-app
```

- request

```bash
zsh ➜ curl 127.0.0.1:80
<h1>Test Homepage</h1>
```

- request if minikube
  - without minikube tunnel, Kubernetes will show the external IP as “pending”

```bash
zsh ➜ minikube tunnel
✅  Tunnel successfully started

📌  NOTE: Please do not close this terminal as this process must stay alive for the tunnel to be accessible ...

❗  The service/ingress my-app requires privileged ports to be exposed: [80]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service my-app.
```

```bash
zsh ➜ curl 127.0.0.1:80
<h1>Test Homepage</h1>
```
