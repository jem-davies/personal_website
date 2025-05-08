---
author: "Jem Davies"
title: "Kubernetes Events -> Bento"
date: "2025-05-08"
description: "Streaming Kubernetes Events to Bento via etcd (with Minikube)"
tags: ["Kubernetes", "etcd", "Bento", "minikube"]
ShowToc: true
ShowBreadCrumbs: false
---

---

## Intro

A while back we had a [request for a "kubernetes events input"](https://github.com/warpstreamlabs/bento/issues/81) for the data streaming tool [Bento](https://warpstreamlabs.github.io/bento/docs/about/) - at the time an [etcd](https://etcd.io/) input connector was being worked on and because kubernetes uses etcd as the default datastore, the etcd input component should enable streaming kubernetes events. 

This blog post walks through the steps to connect a [minikube](https://minikube.sigs.k8s.io/docs/) based local kubernetes cluster's etcd with Bento.

⚠️ WARNING ⚠️

THIS FOR DEV/TESTING ONLY - EXPOSING ETCD LIKE THIS IS A SECURITY RISK

---

## Boot up minikube

```bash
minikube start
```

When you run `minikube start` - your kube config should be updated automatically, meaning that subsequent `kubectl` commands should connect to this minikube cluster.

Let's have a look at what pods we are running:

```bash
kubectl get pods -A
```

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-668d6bf9bc-d5zv4           1/1     Running   0          15s
kube-system   etcd-minikube                      1/1     Running   0          21s
kube-system   kube-apiserver-minikube            1/1     Running   0          21s
kube-system   kube-controller-manager-minikube   1/1     Running   0          21s
kube-system   kube-proxy-8qlpl                   1/1     Running   0          15s
kube-system   kube-scheduler-minikube            1/1     Running   0          21s
kube-system   storage-provisioner                1/1     Running   0          19s
```

When minikube boots up it starts a number of containers in the namespace: `kube-system`, we can see that we have an `etcd-minikube` pod, 
this is the etcd instance used by Kubernetes, that keeps track of changes made to the cluster!

Minikube deploys etcd with mTLS authentication and a self-signed certificate - the next part outlines steps to connect to etcd.

## Grab some certificates from the cluster

In this guide we are going to connect to this from _outside_ of the cluster, to do this we need to grab some tls certs and expose the endpoint to our host machine. When minikube boots up it generates a certificate and key used for health checks - we are going to reuse them for the purpose of this guide. This really isn't what you should be doing in production but this is just a bit of local fun.

Run the following on your local machine (not the cluster) to grab the ca.crt, client.crt and client.key files and save them to your file system:

```bash
minikube ssh -- sudo cat /var/lib/minikube/certs/etcd/ca.crt > ca.crt
minikube ssh -- sudo cat /var/lib/minikube/certs/etcd/healthcheck-client.crt > client.crt
minikube ssh -- sudo cat /var/lib/minikube/certs/etcd/healthcheck-client.key > client.key
```

## Expose etcd's endpoint

We are going to create a [kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/) to expose etcd's endpoint.

```bash
kubectl describe pod etcd-minikube -n kube-system
```

You should see something like: 

```
Name:                 etcd-minikube
Namespace:            kube-system
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 minikube/192.168.49.2
Start Time:           Thu, 08 May 2025 16:35:31 +0100
Labels:               component=etcd
                      tier=control-plane
    ...
```

We are going to use the `Labels` to map our service to this pod, in the `selector` field:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-minikube-service
  namespace: kube-system
spec:
  selector:
    component: etcd
    tier: control-plane
  ports:
    - protocol: TCP
      port: 2379
      targetPort: 2379
  type: NodePort
```

Apply the service to the cluster:

```sh
kubectl apply -f etcd-service.yaml
```

Use `minikube service` to expose it: 


```shell
minikube service etcd-minikube-service -n kube-system
```

you should see: 

```
|-------------|-----------------------|-------------|---------------------------|
|  NAMESPACE  |         NAME          | TARGET PORT |            URL            |
|-------------|-----------------------|-------------|---------------------------|
| kube-system | etcd-minikube-service |        2379 | http://192.168.49.2:31670 |
|-------------|-----------------------|-------------|---------------------------|
🏃  Starting tunnel for service etcd-minikube-service.
|-------------|-----------------------|-------------|---------------------------|
|  NAMESPACE  |         NAME          | TARGET PORT |          URL              |
|-------------|-----------------------|-------------|---------------------------|
| kube-system | etcd-minikube-service |             | http://127.0.0.1:54516    |
|-------------|-----------------------|-------------|---------------------------|
🎉  Opening service kube-system/etcd-minikube-service in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

Now we should be able to connect to etcd on the address: `http://127.0.0.1:54516` (your port will be different, probably)


## Running a bento stream to consume kubernetes events: 

```yaml
input: 
  etcd:
    endpoints:
      - 127.0.0.1:54516 # From the output of minikube service 
    key: "" # setting the key to blank with options.with_prefix set to true will get all the keys
    options:
      with_prefix: true
    tls:
      enabled: true 
      root_cas_file: ./ca.crt # filepath to your certificate authority cert.
      client_certs:
        - key_file: ./client.key # filepath to your key
          cert_file: ./client.crt # filepath to your cert 

output: 
  stdout: {}
``` 

You should see events being printed to the console: 


```json
[
    {
        "create_revision": 383,
        "key": "/registry/services/endpoints/kube-system/k8s.io-minikube-hostpath",
        "lease": 0,
        "mod_revision": 1103,
        "type": "PUT",
        "value": "azhzAAoPCgJ2MRIJRW5kcG9pbnRzEuQDCuEDChhrOHMuaW8tbWluaWt1YmUtaG9zdHBhdGgSABoLa3ViZS1zeXN0ZW0iACokMGNmZmM1YzktMTAxZi00Zjg1LWIzMjAtY2YyZDEzMzBlOTgyMgA4AEIICOie88AGEABi5wEKKGNvbnRyb2wtcGxhbmUuYWxwaGEua3ViZXJuZXRlcy5pby9sZWFkZXISugF7ImhvbGRlcklkZW50aXR5IjoibWluaWt1YmVfYTZlNGRiMTgtNjM5Ny00ZTY1LTg1NGEtMDk4YjI1MTkxZWEyIiwibGVhc2VEdXJhdGlvblNlY29uZHMiOjE1LCJhY3F1aXJlVGltZSI6IjIwMjUtMDUtMDhUMTU6MzY6MDhaIiwicmVuZXdUaW1lIjoiMjAyNS0wNS0wOFQxNTo1MDo1M1oiLCJsZWFkZXJUcmFuc2l0aW9ucyI6MH2KAZQBChNzdG9yYWdlLXByb3Zpc2lvbmVyEgZVcGRhdGUaAnYxIggI3aXzwAYQADIIRmllbGRzVjE6WwpZeyJmOm1ldGFkYXRhIjp7ImY6YW5ub3RhdGlvbnMiOnsiLiI6e30sImY6Y29udHJvbC1wbGFuZS5hbHBoYS5rdWJlcm5ldGVzLmlvL2xlYWRlciI6e319fX1CABoAIgA=",
        "version": 441
    }
]
```

So it is possible to get kubernetes events directly from etcd using Bento BUT - you see that the value is protobuf encoded and I am not sure where the schema files are to decode this. It's probably better to create some new Bento components to read from the kubernetes api anyway... 😅

