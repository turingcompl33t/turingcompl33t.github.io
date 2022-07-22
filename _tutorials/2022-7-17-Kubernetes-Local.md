---
layout: post
title: "Running Kubernetes Locally with Minikube"
---

This tutorial walks through the process of running a simple web service locally with Minikube.

### Install Tools (Prerequisites)

Following along with this tutorial requires:

- [Docker](https://www.docker.com/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

Each of these tools have great installation instructions that can be found in their respective documentations. On my system, the versions in use are as follows:

`docker`:

```bash
Docker version 20.10.17, build 100c701
```

`minikube`:

```bash
minikube version: v1.26.0
commit: f4b412861bb746be73053c9f6d2895f12cf78565
```

`kubectl`:

```bash
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.3", GitCommit:"aef86a93758dc3cb2c658dd9657ab4ad4afc21cb", GitTreeState:"clean", BuildDate:"2022-07-13T14:30:46Z", GoVersion:"go1.18.3", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.4
```

### Building and Publishing a Custom Web Service

This is not a Docker tutorial so we won't spend a significant amount of time working on the web service itself. However, I think it is slightly more satisfying when we can run our own application with Kubernetes, rather than doing so with a pre-built container.

The example we use in this tutorial is a simple Python web server built with `flask`. The source is extremely simple (just 5 lines), and the Dockerfile used to build the web server container does not contain any surprises. 

Once the image is built, we can run the container locally and hit the server to ensure everything is working:

```bash
docker run -p 5000:5000 -d --rm server:latest
wget http://localhost:5000
<p>Hello, World!</p>
```

Push the image to the container registry.

```bash
# Authenticate
aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin public.ecr.aws

# Create a public ECR repository
aws ecr-public create-repository \
    --repository-name ${REPO_NAME} \
    --region ${AWS_REGION}

# Tag the image
docker tag web:latest ${REPO_URI}
# Push to our public repository on ECR
docker push ${REPO_URI}
```

### Start a `minikube` Cluster

```bash
minikube create --nodes 2
...
```

`minikube` does the work of configuring the current context for `kubectl`:

```bash
kubectl config current-context
web-cluster
```

This means that we can immediately interact with our cluster with `kubectl`:

```bash
NAME              STATUS     ROLES           AGE   VERSION
web-cluster       Ready      control-plane   86s   v1.24.1
web-cluster-m02   NotReady   <none>          18s   v1.24.1
```

### Create a Deployment

TODO

### References

- [Set Up Ingress on Minikube with the NGINX Ingress Controller](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)

### Appendix

This section contains some additional steps I took to complete the setup for this tutorial.

**Install Minikube**

These instructions come directly from the [`minikube` setup guide](https://minikube.sigs.k8s.io/docs/start/). They are specific to 64-bit Linux.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Install `kubectl`**

These instructions come directly from the [`kubectl` installation instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/). They are specific to 64-bit Linux.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
