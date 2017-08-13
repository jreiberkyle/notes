# Running docker on AWS

## Scenario

I have a docker container that I am running locally but I want to run it on AWS because of compute limitations locally.

I care about matching my local compute power and keeping costs down. 
My local docker machine has 8Gb memory and I am using 1 CPU. I plan to start the docker-machine instance, upload data,
run the computation, view/download results, and stop the instance. 

## Plan

I want to use docker-machine with the AWS driver to create the AWS instance. This allows me to easily start/stop the
instance from my local command line and generally interact with the instance as I would a local docker machine.

Based on the instance descriptions, an M4 instance would be the best match for my use case. An M4.large instance provides
8Gb memory, the same as my local docker machine. The cost for this instance in US-West (Oregon) is $0.1/hour. The code for US-West (Oregon) is `us-west-2`.

References:
1. [docker-machine with aws example](https://docs.docker.com/machine/examples/aws/#step-2-use-machine-to-create-the-instance)
2. AWS [instance description](https://aws.amazon.com/ec2/instance-types/) and
[ec2 pricing](https://aws.amazon.com/ec2/pricing/on-demand/)
3. [AWS regions](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)

## Implementation

### Set up AWS

Follow steps 1 and 2.1 of [docker-machine with aws example](https://docs.docker.com/machine/examples/aws/#step-2-use-machine-to-create-the-instance)

### Create Docker Machine

```
# spin up an m4.large instance in us-west Oregon
# specify engine-install-url because docker for mac still doesn't have bugfix 
# ref:
# https://forums.docker.com/t/docker-machine-provisioned-aws-instance-can-not-start-docker-engine/34200
# https://github.com/docker/machine/issues/4198#issuecomment-315847971
docker-machine create \
  --driver amazonec2 \
  --amazonec2-region='us-west-2' \
  --amazonec2-instance-type='m4.large' \
  --engine-install-url=https://web.archive.org/web/20170623081500/https://get.docker.com \
  aws-notebook
```

Confirm machine is up and running:

`docker-machine ls`

Since we aren't going to use the machine immediately, stop the instance:

`docker-machine stop aws-notebook`

### Run Container

Workflow:
1. Start Machine
2. Upload data
3. Run container
4. View result
5. Stop Machine
