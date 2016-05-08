# kube2iam

Provide IAM credentials to containers running inside a kubernetes cluster based on annotations.

## Context

Traditionally in AWS, service level isolation is done using IAM roles. IAM roles are attributed through instance 
profiles and are accessible by services through the transparent usage by the aws-sdk of the ec2 metadata API. 
When using the aws-sdk, a call is made to the ec2 metadata API which provides temporary credentials 
that are then used to make calls to the AWS service.

## Problem statement

The problem is that in a multi-tenanted containers based world, multiple containers will be sharing the underlying 
nodes. Given containers will share the same underlying nodes, providing access to AWS
resources via IAM roles would mean that one needs to create an IAM role which is a union of all
IAM roles. This is not acceptable from a security perspective.

## Solution

The solution is to redirect the traffic that is going to the ec2 metadata API for docker containers to a container 
running on each instance, make a call to the AWS API to retrieve temporary credentials and return these to the caller. 
Other calls will be proxied to the ec2 metadata API. This container will need to run with host networking enabled 
so that it can call the ec2 metadata API itself. 

## Usage

### IAM roles

It is necessary to create an IAM role which can assume other roles and assign it to each kubernetes worker.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sts:AssumeRole"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

The roles that will be assumed must have a Trust Relationship which allows them to be assumed by the `root` role. 
See this [StackOverflow post](http://stackoverflow.com/a/33850060) for more details.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### kube2iam daemonset

Run the kube2iam container as a daemonset (so that it runs on each worker) with `hostNetwork: true`.

```
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube2iam
  labels:
    app: kube2iam
spec:
  template:
    metadata:
      labels:
        name: kube2iam
    spec:
      hostNetwork: true
      containers:
        - image: jtblin/kube2iam:latest
          name: kube2iam
          args:
            - "--base-role-arn=arn:aws:iam::123456789012:role/"
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
```

### iptables

To prevent containers to directly access the ec2 metadata API and gain unwanted access to AWS resources, 
the traffic to `169.254.169.254` must be proxied for docker containers.

    iptables -t nat -A OUTPUT -p tcp -d 169.254.169.254 -i docker0 -j DNAT --to-destination `curl 169.254.169.254/latest/meta-data/local-ipv4`:8181

### kubernetes annotation

Add an `iam/role` annotation to your containers with the role that you want to assume for this container.

```
---
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  labels:
	name: aws-cli
  annotations:
	iam/role: role-name
spec:
  containers:
  - image: fstab/aws-cli
	command:
	  - "/home/aws/aws/env/bin/aws"
	  - "s3"
	  - "ls"
	  - "some-bucket"
	name: aws-cli
```

### Options

By default, `kube2iam` will use the in-cluster method to connect to the kubernetes master, and use the `iam/role`
annotation to retrieve the role for the container. Either set the `base-role-arn` option to apply to all roles
and only pass the role name in the `iam/role` annotation, otherwise pass the full role ARN in the annotation.

```
$ kube2iam --help
Usage of kube2iam:
      --api-server string              Endpoint for the api server
      --api-token string               Token to authenticate with the api server
      --app-port string                Http port (default "8181")
      --base-role-arn string           Base role ARN
      --default-role string            Default IAM role (default "default")
      --iam-role-key string            Pod annotation key used to retrieve the IAM role (default "iam/role")
      --insecure                       Kubernetes server should be accessed without verifying the TLS. Testing only
      --log-flush-frequency duration   Maximum number of seconds between log flushes (default 5s)
      --metadata-addr string           Address for the ec2 metadata (default "169.254.169.254")
      --verbose                        Verbose
      --version                        Print the version and exits
```

# Author

Jerome Touffe-Blin, [@jtblin](https://twitter.com/jtblin), [About me](http://about.me/jtblin)

# License

kube2iam is copyright 2016 Jerome Touffe-Blin and contributors. 
It is licensed under the BSD license. See the include LICENSE file for details.