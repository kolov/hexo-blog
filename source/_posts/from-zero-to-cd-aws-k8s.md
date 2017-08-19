---
title: From Zero to Continuous Delivery on Kubernetes on AWS in 2 hours
date: 2017-08-15 21:44:14
tags: aws, kubernetes, circleci, Continuous Delivery
---

For a developer, Kubernetes is a dream target platform. Commands are simple and powerful, making deployment tasks easy and transparent. This is especially valuable in the microservices world, where deployment is something developers have to work with on a daily basis and understand in detail.
Setting up a Kubernetes cluster is a different story, or at least used to be. Kubernetes runs on many platforms, but setting it up is not equally simple. Until recently, any other platform than Google Container Engine required too much operational effort to set up and maintain.

Some time ago I had set up a small cluster on Google Container Engine  (GKE) and a delivery pipeline with CircleCI.
This weekend, I decided to check out how difficult it would be to move this to AWS, and I was surprised how smoothly it went. Starting from a scratch, it should be possible to set up the cluster and a delivery pipeline to it within a few hours. This blog documents the process in big lines.

## The Goal

Starting with any project that produces a docker image, we will create the kubernetes cluster and set up a delivery pipeline that builds and deploys it the cluster on every change.

## Kubernetes on AWS
First, create the cluster. [Kops](https://github.com/kubernetes/kops) has made that really simple.
Choosing the simplest DNS option, a fully operation cluster can be set up with just
the following commands:

{% codeblock lang:shell %}
aws s3api create-bucket --bucket kolov-k8s1-state-store --region us-east-1
aws s3api put-bucket-versioning --bucket kolov-k8s1-state-store --versioning-configuration Status=Enabled
export KOPS_STATE_STORE=s3://kolov-k8s1-state-store
kops create cluster --zones eu-central-1a  kolov-k8s1.k8s.local
{% endcodeblock %}

These commands are more of an example to show how simple it is to create a cluster with `kops`, for the details
follow the official [tutorial](https://github.com/kubernetes/kops/blob/master/docs/aws.md). `kops` sets up the cluster and configures `kubectl` with credentials to access it.

## Container Registry
As we are going to build an deploy containers, we need a container registry. If you already have one somewhere -
great. Setting up one on AWS is pretty straightforward - go to `EC2 Container Service` and create a registry. We need credentials for the build process to push images to it - create a IAM user with `AmazonEC2ContainerRegistryPowerUser` policy. Note its Access Key ID and Secret Key for later use.

## The project

We start with a project that already has a `Dockerfile`, but next to it we need two more files: `k8s-deployment.yml` to create a [kubernetes deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), and `k8s-service.yml` for the [service](https://kubernetes.io/docs/concepts/services-networking/service/).

A typical `k8s-deployment.yml`. It contains a few placeholders:
{% codeblock lang:yaml %}
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: myproject
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app:myproject
        spec:
          containers:
          - name: myproject
            image: %REGISTRY%/myproject:%VERSION%
            ports:
            - containerPort: 80
{% endcodeblock %}
and `k8s-service.yml`:
{% codeblock lang:yaml %}
    apiVersion: v1
    kind: Service
    metadata:
      name: myproject
    spec:
      type: NodePort
      ports:
        - name: myproject
          port: 80
          targetPort: 80
      selector:
        app: myproject
{% endcodeblock %}
## CircleCI

There are  many choices for a CI server, and CircleCI is quite a good one.  They offer a free 1500 hours/month plan - this is 50 building hours a day, isn't that insane? Go to [CircleCI](https://circleci.com/) and create an account if you don't have one already. Your project needs a file named `circleci.yml` that contains the build definition. An example of a Node project
{% codeblock lang:yaml %}
    machine:
      node:
        version: 6.11.2
      services:
        - docker
      environment:
        BUILD_TARGET_DIR: dist
        APP_NAME: myproject

    dependencies:
      cache_directories:
        - node_modules
        - code/server/node_modules
        - code/client/node_modules
        - code/client/vendor
      override:
        - npm install
        - npm install -g bower gulp
        - bower install
        - gulp build

    deployment:
      prod:
        branch: master
        commands:
          - cp docker/nginx/nginx.conf dist
          - cp docker/nginx/Dockerfile dist
          - wget https://raw.githubusercontent.com/kolov/k8s-stuff/master/circleci/deploy-aws.sh
          - chmod +x deploy-aws.sh
          - ./deploy-aws.sh
{% endcodeblock %}
After the regular build, this delegates to a script where the build of the meat is: `deploy-aws.sh`.

## The build script

We are at the last piece of the puzzle. The script has to authenticate to both the container registry and the kubernetes cluster. The key to that is the environment variables feature in CircleCI. Variables can be set, copied between projects, and shared within workflows.
For the registry, define `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. That is enough to execute `aws ecr get-login`, which outputs a docker login command. Two other AWS parameters needed to set up the details are `AWS_ACCOUNT_ID` and `AWS_DEFAULT_REGION`.

The last thing is the `kubectl` authentication. There could be a cleaner way, but as I couldnt find it, I just pass the whole kube configuration file that `kops` created on the local machine to CircleCI: `cat ~/.kube/config | base64 ` and store the output as `KUBECONFIGDATA` in CircleCI.

With all this params, this is the script:
{% codeblock lang:shell %}
		set +x
		curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
		chmod +x kubectl
		mkdir ~/.kube
		echo $KUBECONFIGDATA | base64 --decode --ignore-garbage > ~/.kube/config
		eval $(aws ecr get-login)
		docker build --rm=false -t $AWS_ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com/$APP_NAME:$CIRCLE_SHA1 $BUILD_TARGET_DIR
		REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com
		docker push $REGISTRY/$APP_NAME:$CIRCLE_SHA1
		sed -e s/%VERSION%/$CIRCLE_SHA1/g -e s/%APP_NAME%/$APP_NAME/g -e s/%REGISTRY%/$REGISTRY/g \
        k8s-deployment.yml > k8s-deployment-latest.yml
		./kubectl apply -f k8s-deployment-latest.yml
		./kubectl apply -f k8s-service.yml
{% endcodeblock %}
The script creates a kubernetes deployment file referring to the latest image and updates the cluster with it. Kubernetes does the rest of the magic. It is fascinating to see how it all can be easily integrated in such a transparent way.
