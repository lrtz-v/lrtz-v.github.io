---
template: main.html
tags:
  - Istio
---

# Istio Mesh

## Istio install

- https://istio.io/latest/docs/setup/getting-started/

## Set up with minikube

```bash
zsh ➜ minikube start
zsh ➜ minikube addons enable ingredd
zsh ➜ minikube tunnel
```

## Istio Samples

```bash
zsh ➜ git clone https://github.com/istio/istio.git

zsh ➜ cd istio/samples
```

## Istio HelloWorld

```bash
zsh ➜ cd helloworld
zsh ➜ kubectl apply -f helloworld.yaml
```
