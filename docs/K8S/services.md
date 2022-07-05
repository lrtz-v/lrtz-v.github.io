# Kubernetes 服务、负载均衡与网络

## Deployment && Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: app
      tier: backend
  template:
    metadata:
      labels:
        app: app
        tier: backend
    spec:
      containers:
        - name: app
          image: app:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  type: NodePort
  selector:
    app: app
    tier: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30001
```

- 如上，我们创建了一个名为 app-svc 的 Service；Service 能够将一个接收 port 映射到任意的 targetPort。 默认情况下，targetPort 将被设置为与 port 字段相同的值，并且具有标签 "app=MyApp" 的 Pod 上
- 通过 nodePort 服务调用

```bash
# kubectl get nodes -o wide

NAME             STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION      CONTAINER-RUNTIME
docker-desktop   Ready    master   25m   v1.19.7   192.168.64.4   <none>        Docker Desktop   4.19.121-linuxkit   docker://20.10.3

# curl 192.168.64.4:30001
<h1>Test Homepage</h1>
```

## 通过 LoadBalance 服务调用

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  type: LoadBalancer
  selector:
    app: app
    tier: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

```bash
curl localhost:8080
```

## ExternalName
