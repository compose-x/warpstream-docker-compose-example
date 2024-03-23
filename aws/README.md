
# Deploy to AWS from compose definition

This section guides you through deploying to your AWS Account, using AWS CloudFormation for IaC and ECS to deploy the
services.
Some initial setup is required in order to integrate to your AWS account and use TLS.
This demo also aims to make the endpoints publicly available.

## Pre-requisites

* An AWS Account, `awscli` installed and local profile configured
* Python 3.8+ to install ecs-compose-x
* An active domain in AWS Route53, required to get a certificate via ACM and DNS entries to the brokers

Set the environment variables
* `CLUSTER_ID`: the cluster ID found in WarpStream Control Plane
* `AGENT_POOL_NAME`: the agent pool name generated for your cluster
* `API_KEY`: an API key to the WarpStream control plane. It's recommended to use a different key for each cluster.

## Architecture

## Deployment

From this directory, run. You must have a valid `DOMAIN_NAME` in Route53 for this to work.
Make sure to have met all the previous requirements.

### Install ecs-compose-x

```bash
python3 -m venv compose-x
source compose-x/bin/activate
pip install pip -U; pip install "ecs-compose-x>=1.0.1"
```

If you have never used [ECS Compose-X](https://docs.compose-x.io) before, first, run ```ecs-compose-x init``` which will
* create a S3 bucket to store CFN templates
* initialize the setting for your AWS Account for ECS to work properly

If you have never used ECS before, the IAM service role for it won't exist on your account. To work around that, you can
try creating a new ECS Cluster with the following command:

```bash
aws ecs create-cluster --cluster-name init
aws ecs delete-cluster --cluster init
```

Note that ECS clusters, unlike with EKS, do not cost you anything, in case you forgot to delete it.


Create a new secret with your WarpStream details.

```bash
aws cloudformation deploy \
    --stack-name warpstream-cluster \
    --template-file warpstream_secrets.yaml \
    --parameter-overrides \
    ClusterId=${CLUSTER_ID} AgentPoolName=${AGENT_POOL_NAME} WarpStreamApiKey=${API_KEY} ClusterName=${CLUSTER_NAME}
```

### Deploy

```bash
# First, let's use `render` to make sure everything should work
DOMAIN_NAME=your-active-domain.tld ecs-compose-x render -d templates --format yaml -p warpstream-demo -f docker-compose.yaml

# Run `ecs-compose-x up` to deploy. Alternatively, use `plan` if you want to preview the resources created.
DOMAIN_NAME=your-active-domain.tld ecs-compose-x up -d templates --format yaml -p warpstream-demo -f docker-compose.yaml
```

#### Hints

With ecs-compose-x, you can re-use existing infrastructure to deploy the services to your AWS Account.
For example, if you already have a VPC you wish to re-use, use [x-vpc.Lookup](https://docs.compose-x.io/syntax/compose_x/vpc.html#lookup)

The same can be done for the S3 bucket, the ACM certificate, and generally all resources (although there are exceptions).
