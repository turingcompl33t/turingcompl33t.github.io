---
layout: post
title: "Container Deployment on AWS"
---

This post describes several options for deploying containerized applicatons on AWS and the tradeoffs in terms of usability, scalability, and cost.

### Overview

All major cloud service providers present services for deploying containerized applications. Amazon Web Services (AWS) is no exception. In fact, this post was inspired by a [recent article](https://www.lastweekinaws.com/blog/the-17-ways-to-run-containers-on-aws/) that discusses 17 distinct ways to deploy containers on AWS. With all of these available options, how do we select the one that is most appropriate for our particular application?

As you might expect, there are tradeoffs present across these container deployment offerings. The way these tradeoffs align with the particular requirements of our application 

The methods we will consider include:

- [AppRunner](#apprunner)
- [Lightsail Containers](#lightsail-containers)
- [EC2](#ec2)
- [ECS](#ecs)
- [Lambda](#lambda)

Throughout this post I will use a machine learning inferennce service as an example application, but the concepts are generalizable to other containerized applications. The inference service in this example provides movie recommendations for users over HTTP(S). An example request to the service looks like:

```
curl https://<PATH.TO.HOST>/recommend/<USER_ID>
```

The expected response is a comma-separated list of movie identifiers.

This post is intended for a (relatively) general audience. I provide an appendix at the end of this post with some additional technical details for those who would like to follow along and try out these services themselves.

### Why Containers?

Containerization takes some additional work beyond development of your application. Why should we bother with containers at all? What problem do they solve?



### AppRunner

[AWS AppRunner](https://aws.amazon.com/apprunner/) is a fully managed container deployment service. With AppRunner's high-level API, nearly all of the details of infrastructure management are hidden from us. To deploy a container with AppRunner, the only required piece of information is the URI of the image on ECR. The service also provides some options for more advanced configuration of our deployment, including:
- vCPU and Memory
- Autoscaling Policy
- Health Checks 
- Security
- Networking

However, all of these configuration options come set with reasonable defaults. I found that the entire process of deploying an application on AppRunner takes just 30 seconds to click through, but the subsequent deployment of the service can take some time (~3 minutes).

Once the service is running, AppRunner provides us with a public address that we can use to make inference queries:

```
curl https://pbueku8vmz.us-east-1.awsapprunner.com/recommend/0
```

As the [pricing](https://aws.amazon.com/apprunner/pricing/) page for AppRunner explains, we pay for both compute and memory resources used by our application.

**Benefits**
- The setup for AppRunner is simple and can be completed quickly, without prior experience in cloud computing technologies.
- The AppRunner service provides builtin auto-scaling and load-balancing. This makes scaling deployments of containers on AppRunner extremely simple.

**Drawbacks**
- The configurability of the service is low. AppRunner does not allow fine-grained control of your infrastructure.

### Lightsail Containers

### EC2

AWS [Elastic Compute Cloud](https://aws.amazon.com/ec2/) (EC2) is the primary infrastructure-as-a-service offering on AWS. This means that with EC2, we are given low-level access to the cloud infrastructure, instead of being presented with a high-level interface to a platform or service offered by AWS.

**Benefits**
- EC2 

**Drawbacks**
- Setting up a container deployment on EC2 requires manual infrastructure provisioning and configuration. Some of these steps can be automated through infrastructure management tools like [Terraform](https://www.terraform.io/).
- As an IaaS offering, EC2 does not automatically provide higher-level functionality such as autoscaling and load-balancing. Scaling a container deployment on EC2 therefore requires significantly more manual effort than other services with a higher-level API.

### ECS

### Lambda

### Appendix: Preparing a Container

This section provides some additional "technical" background on preparing a container for deployment using the services presented in this post. Specifically, we'll look at building a container image for our application, and then uploading this image to the AWS elastic container registry (ECR).

The full setup for a containerized application is beyond the scope of this post, but there are many existing tutorials online that walk through this procedure. Here, we will assume that we already have a functioning application that uses a trained machine learning model to perform real-time inference. From here, we need to perform the following high-level steps to build a container image for our application:

- Prepare an appropriate base image (i.e. one that has all of the underlying dependencies)
- Move the source code, trained model, and requirements to the container
- Install inference dependencies in the container
- Run the server

The following dockerfile implements all of these steps, with some comments that identify how they are related to the steps above.

```Dockerfile
# Prepare the base image; there are many appropriate choices here!
FROM python:3.8-slim-buster
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential

# Use /home as the working directory
WORKDIR /home

# Copy the source code to the container FS
ADD src/ /home/src
# Copy the trained model to the container FS
ADD trained_models/ /home/trained_models
# Copy the python requirements to the container FS
ADD requirements.txt /home

# Install Python requirements
RUN pip install -r requirements.txt

# The server listens on port 8082 for incoming connections
EXPOSE 8082

# Run the server
CMD ["python", "src/app.py", "trained_models/"]
```

With this setup, we can run the following command to build our image locally with the `inference` tag:

```bash
docker build --tag inference .
```

Now that the container is built, we can upload it to [ECR](https://aws.amazon.com/ecr/). The following commands assume that you have the following environment variables set in your shell session:

```bash
export AWS_ID="<YOUR_ACCOUNT_ID>"
export AWS_REGION="<YOUR_AWS_REGION>"
export REPO_NAME="<YOUR_REPO_NAME>"
```

First we need to to authenticate the Docker client on our local machine with AWS:

```bash
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

This command notifies us when authentication succeeds. Now we can create the ECR repository that will store our images:

```bash
aws ecr create-repository --repository-name ${REPO_NAME}
```

Finally, we push the image to the repository:

```bash
# Tag the image
docker tag inference:latest ${AWS_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:inference-latest
# Push to our private repository on ECR
docker push ${AWS_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:inference-latest
```

Now our container image is available at `<AWS_ID>.dkr.ecr.us-east-1.amazonaws.com/<REPO_NAME>:inference-latest` and is ready for deployment via the services explored throughout the rest of this post.

### References

- [The 17 Ways to Run Containers on AWS](https://www.lastweekinaws.com/blog/the-17-ways-to-run-containers-on-aws/)