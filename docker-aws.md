# Running docker on AWS

## Scenario

I have a docker container that I am running locally but I want to run it on AWS because of compute limitations locally. This container runs an application that performs Cross-Validation to determine the best number of neighbors, k, in a KNN Classifier. The training data is saved as `xy_files.npz` and cross-validation is performed using the `scikit-learn` library.

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

Create docker machine instance on AWS:

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

Set new machine as default:

`eval $(docker-machine env aws-notebook`

Confirm machine is up and running:

`docker-machine ls`

Get machine IP:

`docker-machine ip aws-notebook`

Ensure docker is running container on aws machine:

```
# Port 8888 is open to inbound connections by the driver, to view/edit
# go to 'docker-machine' security group in AWS
docker run -d -p 8888:80 --name webserver kitematic/hello-world-nginx
```

Point browser to: https://<machine ip>:8888. Page should say 'Voilà! Your nginx container is running!'

Shut down/halt container:

```
docker stop webserver
docker rm webserver
```

Since we aren't going to use the machine immediately, stop the instance:

`docker-machine stop aws-notebook`

### Run Container

Workflow:
1. Start Machine and get IP
```
docker-machine start aws-notebook
# Because the instance will have a new IP, regenerate certs
docker-machine regenerate-certs -f aws-notebook
eval $(docker-machine env aws-notebook)
docker-machine ip aws-notebook
```
1. Upload data (use [docker-machine scp](https://docs.docker.com/machine/reference/scp/))
```
docker-machine scp data/knn_cross_val/xy_file.npz aws-notebook:/home/ubuntu/xy_file.npz
```
1. Build image
The docker image needs `numpy` and `scikit-learn` and needs to add the cross-val directory.

`Dockerfile.crossval`:
```
# Cross-validation only requires numpy and scikit-learn,
# both of which are installed in the base image 
FROM jupyter/scipy-notebook
```

```
docker build -f Dockerfile.crossval -t crossval .
```

1. Run container
1. View result
1. Stop Machine
