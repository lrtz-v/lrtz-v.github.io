<!-- ---
title: Kubernetes服务、负载均衡与网络
date: 2021-01-08 10:03:01
tags: [k8s, Kubernetes, kubernetes]
--- -->

# Kubernetes服务、负载均衡与网络

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-server
spec:
  selector:
    app: app
  ports:
    - name: default
      protocol: TCP
      port: 8080
      targetPort: 8080
```

- 如上，我们创建了一个名为 my-app-server 的 Service；Service 能够将一个接收 port 映射到任意的 targetPort。 默认情况下，targetPort 将被设置为与 port 字段相同的值，并且具有标签 "app=MyApp" 的 Pod 上

### 没有选择算符的 Service

 ```yaml
 apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
 ```

- 由于此服务没有选择算符，因此 不会 自动创建相应的 Endpoint 对象
- 可以通过手动添加 Endpoint 对象，将服务手动映射到运行该服务的网络地址和端口

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```
