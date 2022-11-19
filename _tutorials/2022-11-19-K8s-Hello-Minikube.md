---
layout: post
title: "K8s: Hello Minikube"
---

This post walks through running your first pods on a local Kubernetes cluster deployed via `minikube`. This tutorial follow the outline of the [official tutorial](https://kubernetes.io/docs/tutorials/hello-minikube/).

### Prerequisites

This tutorial assumes that you have the following programs installed:

- `docker` (see [here](https://docs.docker.com/engine/install/ubuntu/))
- `minikube` (see [here](https://minikube.sigs.k8s.io/docs/start/))
- `kubectl` (see [here](https://kubernetes.io/docs/tasks/tools/))

The version information for the tools used in this tutorial is as follows:

`docker`:

```bash
$ docker --version
Docker version 20.10.21, build baeda1f
```

`minikube`:

```bash
$ minikube version
minikube version: v1.26.0
commit: f4b412861bb746be73053c9f6d2895f12cf78565
```

`kubectl`:

```bash
$ kubectl version
Client Version: v1.24.3
Kustomize Version: v4.5.4
```

### Create a Minikube Cluster

Start a single-node K8s cluster with `minikube`:

```bash
minikube start
```

To open the K8s dashboard provided by `minikube`, run:

```bash
minikube dashboard
```

### Create a Deployment

Now that the K8s cluster is running, we can deploy a _pod_ to the cluster with `kubectl`. A _deployment_ is the preferred method for deploying and managing the lifecycle of pods. For this pod, we use a container image provided by the K8s organization (`registry.k8s.io/e2e-test-images/agnhost:2.39`). Once deployed to the cluster, the single-container pod implements a simple web service.

Deploy the pod to the cluster with `kubectl`:

```bash
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
```

We can now use `kubectl` to query information about our deployment / pod:

```bash
# View the deployment
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           40s
# View the pod
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-667997dc6c-fqvrl   1/1     Running   0          44s
```

This information is also available in the web-based dashboard provided by `minikube`.

### Create a Service

Now we have a pod running in the cluster, but we have no way of interacting with the web service that it implements. In order to expose the web service to requests from outside the K8s virtual network, we need a _service_.

Expose the port on which the webserver runs with `kubectl`:

```bash
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
```

Under the hood, the `kubectl expose` command creates a service that exposes the webserver port outside of the cluster. We can check the service that was created with:

```bash
kubectl get services
```

At this point, if our cluster were not deployed with `minikube`, an external IP address would be allocated and our webservice would be accessible there. With `minikube`, we have to user the `minikube service` command to access the newly exposed service:

```bash
minikube service hello-node
```

### Cleanup

Cleanup our service and deployment with:

```bash
kubectl delete service hello-node
kubectl delete deployment hello-node
```

### References

- [Kubernetes Official: Hello Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)
- [`minikube` Documentation](https://minikube.sigs.k8s.io/docs/)
- [`kubectl` Documentation](https://kubernetes.io/docs/reference/kubectl/)
- [`docker` Documentation](https://docs.docker.com/engine/)