---
layout: post
title: "Provisioning Compute with a Custom Virtual Machine Image on AWS EC2"
---

This post walks through all of the steps necessary to setup a minimally-intrusive local development environment for interaction with AWS via the AWS CLI and subsequently use this setup to provision an EC2 compute instance from a custom AMI.

### Prerequisites / Assumptions

The only software prerequisite for this tutorial is Docker (accessible via the `docker` command-line interface). We assume a *NIX-based development environment (I'm on Ubuntu Linux 18.04), but extension to Windows should prove relatively straightforward. 

### Motivation

A class in which I am currently enrolled (CMU's 15-745: _Optimizing Compilers_) makes extensive use of the LLVM compiler infrastructure. Because configuration of a local install of LLVM may vary considerably among the development environments of all of the students in the class, the course staff require that all projects build and run successfully in a specific virtual machine image that they provide. While they recommend using a local virtualization solution such as Oracle's VirtualBox to run and develop within the VM locally, I was wary of setting up VirtualBox on my workstation because of prior unpleasant experiences with the software. I figured a simple alternative would be to spin up a compute instance on AWS EC2 and run the provided image there. While this approach ultimately proved successful, it required a more involved setup procedure than I anticipated. 

The majority of the content in this post is available on one or another of Amazon's official documentation pages, but I believe it will be beneficial for others who want to achieve something similar to have the entirety of the process described in one place.

### Setting Up: The AWS CLI

The first step to provisioning a new EC2 instance based on a custom VM image is to setup the AWS CLI tool. At first I hoped that this would not be necessary because the whole point of this exercise was to avoid installing additional software on my personal workstation, but I was unable to locate instructions for importing a VM image into an AMI via any means other than the CLI.

With that said, I was not ready to admit defeat, so instead of installing the AWS CLI tool locally as the documentation suggests, I opted to use the [official AWS CLI Docker image](https://hub.docker.com/r/amazon/aws-cli) to run the tool locally within a Docker container. 

Setting up the AWS CLI within a Docker container is refreshingly simple. First, we need to pull the latest version of the official AWS CLI docker image from DockerHub:

```
$ docker pull amazon/aws-cli:latest
```

We can then run arbitrary `aws` commands in the container like so:

```
$ docker run --rm -it amazon/aws-cli:latest <COMMAND>
```

This approach works fine for one-off commands, but generally it would be more efficient to drop into a shell in the context of the container and run commands interactively within it. However, it doesn't appear that the official AWS CLI docker image provides interactive functionality (this might warrant further investigation). A simple modification that addresses the verbosity of the command is the establishment of a shell alias for the `aws` command that transparently invokes the desired command within a Docker container:

```
alias aws='docker run --network="host" --rm -ti -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli'
```

Additionally, if you require super-user privileges to run `docker` on your system (likely), you can prefix the aliased command with `sudo` to avoid typing it every time:

```
alias aws='sudo docker run --network="host" --rm -ti -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli'
```

This alias is the same as that found in the AWS documentation, apart from the fact that I found it necessary to add the `--network="host"` to the above alias in order to successfully reach the AWS API. An important thing to notice is the fact that this command maps the `~/.aws` directory on our local filesystem to the `/root/.aws` directory within the container. This feature will prove important later in this tutorial. 

We can verify that the alias is working as expected by checking the version of the AWS CLI tool installed within the container:

```
$ aws --version
aws-cli/2.1.24 Python/3.7.3 Linux/5.4.0-65-generic docker/x86_64.amzn.2 prompt/off
```

However, this only tests "local" functionality of the CLI. To verify that we can properly connect to and authenticate with AWS, we need to run a command that interacts with one of the AWS services. In order to do this though, we will need to authenticate with AWS via the CLI.

If you do not already have an IAM user on AWS with which you can authenticate you should create one now. The web interface to this functionality can be found under the _Users_ tab on the [IAM service page](https://console.aws.amazon.com/iam/home/) of the AWS web console. The IAM permissions required to import a VM are listed in the [official AWS documentation](https://docs.aws.amazon.com/vm-import/latest/userguide/vmie_prereqs.html#iam-permissions-image). Once the IAM user is created and the keys are available, we can configure the CLI to use these credentials when authenticating with AWS by running the `aws configure` command.

With authentication established, we can verify connectivity with the AWS API by running a command like:

```
$ aws ec2 describe-instances
```

If you have any active instances on EC2, this should produce a JSON result describing each of them.

While I am largely satisfied with this approach to interacting with AWS via the CLI, it introduces a few obvious limitations. Perhaps the most noticeable is the lack of support for the auto-completion / suggestion functionality that the `aws` tool provides when invoked directly. Apart from this, I have also observed that using Docker introduces (sometimes less-than-negligible) latency when issuing individual commands, but this is the unavoidable price we pay for invoking the tool within a container in the first place.

### Creating a Custom AWS AMI

Now that we are able to interact with AWS via the command line tool, we can move on to the task at hand: creating a custom AMI with which we can provision new EC2 compute instances.

**Preparing the Image**

The first and most straightforward step in creating the custom AMI from a virtual machine image is to upload the image of interest to S3. This step is relatively trivial, and is described well in the [AWS documentation](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#upload-image), so I won't reiterate it here.

**Creating the IAM Role**

After the image is uploaded to S3, the next step is to create a new service role, `vmimport`, that will be permitted to perform the VM import and export operations on our behalf. Creating this role requires a couple distinct steps, the first of which is create the role itself with the `create-role` command. We define the role via a JSON file `trust-policy.json` as follows:

```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```

We have a slight issue though: if we try to provide the path to this file as it exists on our local filesystem, the AWS CLI will not be able to locate it as it runs in the context of the container. We can circumvent this issue by noticing that the `alias` command we established above maps the `~/.aws` directory of the local filesystem to `/root/.aws` within the container. Therefore, all we need to do to point the `aws` tool to the correct location of the file is move the `trust-policy.json` file to the `~/.aws` directory; for the purposes of this tutorial, I create a subdirectory within `~/.aws` called `vmimport` (don't let this confuse you: the name of the directory has nothing to do with the `vmimport` role that we are currently creating) in which I will place all of the required files.

Now, when we invoke the `create-role` command via `aws`, we point it at the location of the `trust-policy.json` file as it will appear within the container:

```
$ aws iam create-role --role-name vmimport --assume-role-policy-document "file:///root/.aws/vmimport/trust-policy.json"
```

Upon invoking this command, we should receive some JSON in response:

```
{
  "Role": {
    "Path": "/",
    "RoleName": "vmimport",
    "RoleId": "...",
    "Arn": "arn:aws:iam::...:role/vmimport",
    "CreateDate": "2021-02-06T23:43:50+00:00",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                  "Service": "vmie.amazonaws.com"
              },
              "Action": "sts:AssumeRole",
              "Condition": {
                  "StringEquals": {
                      "sts:Externalid": "vmimport"
                  }
              }
            }
        ]
    }
  }
}
```

Now we need to create a policy and attach it to the role that we have just created. We define the policy in the file `role-policy.json`:

```
{
  "Version":"2012-10-17",
  "Statement":[
     {
        "Effect": "Allow",
        "Action": [
           "s3:GetBucketLocation",
           "s3:GetObject",
           "s3:ListBucket" 
        ],
        "Resource": [
           "arn:aws:s3:::<VM IMPORT BUCKET>",
           "arn:aws:s3:::<VM IMPORT BUCKET>/*"
        ]
     },
     {
        "Effect": "Allow",
        "Action": [
           "s3:GetBucketLocation",
           "s3:GetObject",
           "s3:ListBucket",
           "s3:PutObject",
           "s3:GetBucketAcl"
        ],
        "Resource": [
           "arn:aws:s3:::<VM EXPORT BUCKET>",
           "arn:aws:s3:::<VM EXPORT BUCKET>/*"
        ]
     },
     {
        "Effect": "Allow",
        "Action": [
           "ec2:ModifySnapshotAttribute",
           "ec2:CopySnapshot",
           "ec2:RegisterImage",
           "ec2:Describe*"
        ],
        "Resource": "*"
     }
  ]
}
```

We then use the `put-role-policy` command to attach the policy from `role-policy.json` to the role created above, observing the same procedure of moving the policy description file to `~/.aws/vmimport` prior to invoking the command:

```
$ aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file:///root/.aws/vmimport/role-policy.json"
```

On successful invocation, this command produces no output.

**Import the Image**

Finally, we have completed all of the steps necessary to actually import the VM image. We accomplish this by first creating a `containers.json` file that describes the image we wish to import:

```
[
  {
    "Description": "LLVM Hacking VM",
    "Format": "ova",
    "UserBucket": {
        "S3Bucket": "<BUCKET_NAME>",
        "S3Key": "<IMAGE_NAME>.ova"
    }
}]
```

And then its as easy as invoking the `import-image` command from the CLI:

```
aws ec2 import-image --description "LLVM Hacking VM" --disk-containers "file:///root/.aws/vmimport/containers.json"
```

This command produces some JSON output that resembles the following:

```
{
    "Description": "LLVM Hacking VM",
    "ImportTaskId": "import-ami-XXXX",
    "Progress": "1",
    "SnapshotDetails": [
        {
            "Description": "LLVM Hacking VM",
            "DiskImageSize": 0.0,
            "Format": "OVA",
            "UserBucket": {
                "S3Bucket": "<BUCKET_NAME>",
                "S3Key": "<IMAGE_NAME>.ova"
            }
        }
    ],
    "Status": "active",
    "StatusMessage": "pending"
}
```

At first I was concerned when I saw the above output because it reports `0.0` for the size of the disk image, and I assumed that something had gone wrong. However, this proved to be merely a temporary inconsistency that was resolved in future status updates, which we can periodically request with the following command:

```
aws ec2 describe-import-image-tasks --import-task-ids import-ami-XXXX
```

where `import-ami-XXXX` is replaced with the string from the `ImportTaskId` from the output of the `import-image` command. For me, the process of importing the image completed in about 10 minutes.

Once the `describe-import-image-tasks` command reports that the import is complete, we should be able to locate our custom AMI in one of the dropdowns when provisioning a new EC2 instance (assuming you do so via the web console).

Congratulations! We now have a workflow for provisioning compute instances from arbitrary VM images on AWS EC2. The natural next-step is to attempt automating this process via one of the many programmatic APIs that AWS exposes (e.g. Python), but I'll save this topic for a (potential) future post.

### References / Further Reading

- [AWS Documentation: VM Import / Export](https://docs.aws.amazon.com/vm-import/latest/userguide/vmie_prereqs.html)
- [AWS Documentation: AWS CLI Docker Image](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-docker.html)