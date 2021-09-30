# Cruncher

This is a simple demo app to demonstrate cluster autoscaling on EKS.

A `queue-watcher` deployment is created, which monitors an SQS queue. When messages appear in the queue, the deployment:

- receives a message from the queue
- creates a Kubernetes job to process it
- deletes the message from the queue

If more resources are needed, the cluster autoscaler kicks in to add more nodes.

## Summary of resources defined in this repo

The `./demo-cluster` directory contains the definitions for an EKS cluster, node group, associated IAM roles/policies, etc. This directory need only be instantiated once.

The `./app` directory contains everything needed to run one instantiation of the "app". For example, to create a `staging` and `production` environment, you could run:

```
terraform workspace new staging
make plan
make apply

terraform workspace new production
make plan
make apply
```

Each Terraform workspace corresponds to a Kubernetes namespace, a dedicated SQS queue and `queue-watcher` deployment, and IAM roles with permissions scoped to these resources.

## Setup

### Dependencies

- Terraform
- AWS CLI
- `jq`
- `make`

To install Terraform:

```
make deps
```

### Set environment variables

Run the following commands to retrieve your account ID and region:

```
aws sts get-caller-identity
aws configure get region
```

Ensure the following environment variables are set:

- `AWS_REGION`
- `AWS_ACCOUNT_ID`
- `MAKEFILES=../Makefile` (to be able to run `make` in subdirectories)
- `WORKSPACE=production` (or another value of your choice)

Run `make env` to show the current values of these variables.

### Create the cluster

In `./demo-cluster/`:

```
terraform init
terraform workspace new demo-cluster
make plan
make apply
```

To be able to connect to nodes, set `TF_VAR_public_key` to the desired public key (in OpenSSH format), then plan/apply.

### Instantiate the demo app

In `./app/`:

#### Build and push queue-watcher image

```
export REGISTRY_SLASH=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
export COLON_TAG=:${WORKSPACE}
aws ecr get-login-password | docker login --username AWS --password-stdin ${REGISTRY_SLASH}
docker-compose build && docker-compose push
```

#### Deploy

```
terraform init
terraform workspace new ${WORKSPACE}
make plan
make apply
```

### Interact with the demo app

Update your kube config:

```
aws eks update-kubeconfig --name demo
kubectl config set-context demo --namespace demo-${WORKSPACE}
# ^ WORKSPACE should match the Terraform workspace in ./app
```

Add some messages to the queue (ensure `AWS_REGION` and `AWS_ACCOUNT_ID` are set):

```
./messages.sh --add 1
```

The values of `TARGET` and `DURATION` environment variables affect the resulting queue messages:


```
TARGET=example.com DURATION=60 ./messages.sh --add 1  # this job will ping example.com for 60 seconds
TARGET=invalid DURATION=10 ./messages.sh --add 1      # this job will fail
```

### Requeue failed jobs

To re-add failed tasks to the queue and delete the failed jobs, run:

```
./requeue-failed.sh
```

### Reset jobs and queue

To purge the queue and all jobs, run:

```
./messages.sh --purge
```

### Behold the autoscaling

As more messages are added to the queue, new nodes should be created.

To list nodes:

```
make nodes-show
```

To view autoscaling activity:

```
make asg-list
make asg-activity ASG=<id from previous command>
```
