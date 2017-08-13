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
8Gb memory, the same as my local docker machine. The cost for this instance in US-West (Oregon) is $0.1/hour.

References:
1. [docker-machine with aws example](https://docs.docker.com/machine/examples/aws/#step-2-use-machine-to-create-the-instance)
2. AWS [instance description](https://aws.amazon.com/ec2/instance-types/) and
[ec2 pricing](https://aws.amazon.com/ec2/pricing/on-demand/)
