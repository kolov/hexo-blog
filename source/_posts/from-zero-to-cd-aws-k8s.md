---
title: From Zero to Continouous Delivery on Kubernetes on AWS in 2 hours
date: 2017-08-15 21:44:14
tags:
---

# From Zero to Continuous Delivery to Kubernetes on AWS in 2 hours

For a developer, Kubernetes is a dream platform to develop for. 
Especially in the microservices world, it is great to have acces to its simple powerful API. 
Setting up a Kubernetes cluster is a different story, or at least used to be. 
In my experience, any other platform than Google Container Engine required too much operational effort to set up and maintain. 

Some time ago I had set up a small cluster on GKE with CircleCI deploying live. 
<a href="https://github.com/kubernetes/kops">Kops</a> claims to make using AWS as simple as GKE, 
so this weekend I decided to move a GKE cluster, including configurati and applications to AWS. 

## The Goal
All wen need in the beginning ia an existing project that builds a docker image. We will create the cubernetes 
cluster 
and will setup a process that builds a docker image and deploys it to the cluster on every change.

## Kubernetes on AWS
First, create the cluster. Coosing the simplest DNS option, a fully operation cluster can be set up with just 
the following commands:

      aws s3api create-bucket --bucket kolov-k8s1-state-store --region us-east-1
      aws s3api put-bucket-versioning --bucket kolov-k8s1-state-store --versioning-configuration Status=Enabled
      export KOPS_STATE_STORE=s3://kolov-k8s1-state-store 
      kops create cluster --zones eu-central-1a  kolov-k8s1.k8s.local

The commands above are more of an example to show how simple it is to create a cluster with `kops`, for the details 
follow the official
 instructions.
 OK, now we have a cluster, and we clould start manually rinning `kubectl` to run our application there, but 
doing that manually is not cool. 

## Container Registry
As we are going to build an deploy containers, we need a container registry. 
If you have one setup somwewhere - great. Setting up one on AWS is pretty straightforward - go to `EC2 Container 
Service` and create the repository. The build process need push access to that ragistry - create a IAM user with 
`arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser` policy. Note its Access Key ID and Secret Key for use by 
the build process.

## The project

A project that build a docker file needs:
* Dockerfle
* Kubernetes deployment descriptor
* Kubernetes servcie descriptor

While `Dockerfile` is specific to the projects, here is a sample `k8s-deployment.yml`:

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: %APP_NAME%
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: %APP_NAME%
        spec:
          containers:
          - name: %APP_NAME%
            image: %REGISTRY%/%APP_NAME%:%VERSION%
            ports:
            - containerPort: 80
            
`k8s-service.yml`:
            
    apiVersion: v1
    kind: Service
    metadata:
      name: curriculi-fe
    spec:
      type: NodePort
      ports:
        - name: curriculi-fe
          port: 80
          targetPort: 80
      selector:
        app: curriculi-fe

## CircleCI

Certainly there are  many choices for a CI server, but we want to be redy in one hour - go directly to 
CircleCI and create an account if you don't have one already. They
offer a free 1500 hours/month plan - this is 50 building hours a day, isn't that insane? After you have a account, 
select a project to build. A file 
named `circleci.yml` defined the build. An example of a Scala project

    machine:
      node:
        version: 6.11.2
      services:
        - docker
      environment: 
        BUILD_TARGET_DIR: dist
        APP_NAME: curriculi-fe
    
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
        
## The build script


		set +x
		curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
		chmod +x kubectl
		mkdir ~/.kube
		echo $KUBECONFIGDATA | base64 --decode --ignore-garbage > ~/.kube/config
		eval $(aws ecr get-login)
		docker build --rm=false -t $AWS_ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com/$APP_NAME:$CIRCLE_SHA1 $BUILD_TARGET_DIR
		REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com
		docker push $REGISTRY/$APP_NAME:$CIRCLE_SHA1
		sed -e s/%VERSION%/$CIRCLE_SHA1/g -e s/%APP_NAME%/$APP_NAME/g -e s/%REGISTRY%/$REGISTRY/g k8s-deployment.yml > k8s-deployment-latest.yml
		./kubectl apply -f k8s-deployment-latest.yml
		./kubectl apply -f k8s-service.yml
		
 