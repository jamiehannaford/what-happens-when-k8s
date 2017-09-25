# What happens when ... Kubernetes

Imagine I want to run nginx in a Kubernetes cluster and make it visible to the outside world. We can do this in a one-liner with kubectl:

```bash
kubectl run --image=nginx --replicas=3 --port=80 --expose
```

But what _really_ happens when you run that command?

One of the beautiful things about Kubernetes is that it offers tremendous power while abstracting complexity through user-friendly interfaces. In order to understand this complexity (and therefore what value Kubernetes offers us), we need to follow the path of a request as it travels through a Kubernetes sytem. This repo seeks to explain that lifecycle.

This is a living document. Contributions that add, edit, or delete content where necessary is definitely welcome!

## api version reconciliation

## creds

## http request to api

## authentication

## authorization

## admission controllers

## object topology created (pods -> RS -> deployment)

## each object save to etcd

## scheduler assigns node

## kubelet deploys container

## create service

## steps authn->etcd repeated

## endpoint created

## kube-proxy writes iptables rules

## dns server adds A/SRV records