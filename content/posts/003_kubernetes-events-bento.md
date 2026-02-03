---
author: "Jem Davies"
title: "Kubernetes Events -> Bento: Part 2"
date: "2026-01-21"
description: "Streaming Kubernetes Events to Bento via k8s API"
tags: ["Kubernetes", "Bento", "minikube"]
ShowToc: true
ShowBreadCrumbs: false
---

---

So a while ago I wrote a post about consuming kubernetes events using Bento's 
[`etcd`](https://warpstreamlabs.github.io/bento/docs/components/inputs/etcd) 
component. Whilst that _could_ be possible, it certainly isn't ideal.

Since then we have had a great contribution with the [`kubernetes_watch`](https://warpstreamlabs.github.io/bento/docs/components/inputs/kubernetes_watch)
input component. 

Here's a quick guide to spinning it up to serve as a 'Part 2' to [Kubernetes Events -> Bento](./001_etcd-kubernetes-bento.md)!

---

### Boot up minikube 

```sh
minikube start
```

`minikube start` will start a minikube cluster & update your kube config, in the default location of `~/.kube/config`.
Bento will look for the kube config in this default location - and use it to connect to the k8s API.

### Create the Bento config

```yaml
# config.yaml
input:
  kubernetes_watch:
    resource: "pods"
    label_selector:
      app: mooit

output:
  stdout: {}
```

We need to specify a resource type to watch, and we also specify a `label_selector` to receive events for the 'mooit' 
application.

We will output the event to `stdout` - although there are many options listed [here](https://warpstreamlabs.github.io/bento/docs/components/outputs/about#categories).

Run Bento:

```sh
bento -c config.yaml
```

### Apply a new deployment

```yaml
# mooit.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mooit-yaml
  labels:
    app: mooit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mooit
  template:
    metadata:
      labels:
        app: mooit
    spec:
      containers:
      - name: mooit
        image: bgulla/mooit
```

```sh
kubectl apply -f mooit.yaml
```

And then in the terminal running Bento you should see some K8s events that look
like this: 

```json
{
    "conditions": [
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2026-01-21T17:26:43Z",
            "status": "True",
            "type": "PodReadyToStartContainers"
        },
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2026-01-21T17:26:43Z",
            "status": "True",
            "type": "Initialized"
        },
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2026-01-21T17:26:58Z",
            "status": "True",
            "type": "Ready"
        },
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2026-01-21T17:26:58Z",
            "status": "True",
            "type": "ContainersReady"
        },
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2026-01-21T17:26:43Z",
            "status": "True",
            "type": "PodScheduled"
        }
    ],
    "containerStatuses": [
        {
            "containerID": "docker://25c9d098d420f66712c4abaa0e60a43ab94550c910e3d4cf46ba30dc0d8977df",
            "image": "registry.k8s.io/kube-apiserver:v1.32.0",
            "imageID": "docker-pullable://registry.k8s.io/kube-apiserver@sha256:ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b",
            "lastState": {},
            "name": "kube-apiserver",
            "ready": true,
            "restartCount": 0,
            "started": true,
            "state": { "running": { "startedAt": "2026-01-21T17:26:40Z" } }
        }
    ],
    "hostIP": "192.168.49.2",
    "hostIPs": [{ "ip": "192.168.49.2" }],
    "phase": "Running",
    "podIP": "192.168.49.2",
    "podIPs": [{ "ip": "192.168.49.2" }],
    "qosClass": "Burstable",
    "startTime": "2026-01-21T17:26:43Z"
}
```

--- 

That's it - much easier than attempting to read from `etcd` directly although that 
was still fun to do. 

There are many options to peruse on the [`kubernetes_watch`](https://warpstreamlabs.github.io/bento/docs/components/inputs/kubernetes_watch) 
input component. 