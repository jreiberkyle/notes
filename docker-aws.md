# Parameter Tuning with Jupyter Notebook on AWS using Docker

## Scenario

I have a docker container that I am running locally but I want to run it on AWS because of compute limitations. This container runs an application that performs Cross-Validation to determine the best number of neighbors, k, in a KNN Classifier. The training data is saved as `xy_files.npz` and cross-validation is performed using the `scikit-learn` library.

I care about matching my local compute power and keeping costs down. 
My local docker machine has 8Gb memory and I am using 1 CPU with 4 cores. I plan to start the docker-machine instance, upload data,
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

### Create AWS CloudWatch Alarm

Since we don't want the docker-machine instance to continue running once it has finished processing
(or if we forget to shut it down), we will create a CloudWatch alarm that will shut it down automatically
after the CPU utilization has been below a threshold for a reasonable duration. The duration should be short
enough to minimize unnecessary cost due to an inactive, running instance but long enough to minimize cost due
to multiple start/stop cycles and the AWS minimum charge unit, 1 hour. I chose a CPU utilization of less than 5% for 45 minutes.

The docker-machine instance can be found in the [US West 2 ec2 console](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2#Instances:sort=instanceId). Select the instance, bring the 'Monitoring' tab to the front, then click the 'Create Alarm' button. In the created alarm, make sure the 'Stop this instance' action is selected and set the case to 'Average' of 'CPU Utilization' is: '<=' '5' Percent (be sure to select 'less than', not 'greater than', which is the default) For at least '9' consecutive periods of '5 Minutes'. Be sure to treat missing data as 'good' so the alarm won't keep on
shutting down the notebook if you just start it up and don't ramp up a process immediately (or at least you will have 45 minutes before this happens).


### Run Container

NOTES:
- The docker image for this container needs ipython notebook and the
`numpy` and `scikit-learn` python libraries. This is provided out-of-the box
in the jupyter/scipy-notebook image. Therefore, we do not need to build our own
image.
- This application only uses one CPU core, while more are available in the instance.
This is because I kept on running into errors when attempting to enable multiprocessing.
Ipython notebook and the `multiprocessing` library (used by `joblib`, which handles
parallel processing in scikit-learn) are not compatible and this results in the process hanging.
When I ran the process as a script on the command-line, I got a memory error.
To enable `joblib` multiprocessing in docker, add `-e JOBLIB_TEMP_FOLDER="/home/jovyan/work/tmp"`
to the `docker run` command.

Workflow:

#### Start Machine and get IP

```
docker-machine start aws-notebook
# Because the instance will have a new IP, regenerate certs
docker-machine regenerate-certs -f aws-notebook
eval $(docker-machine env aws-notebook)
docker-machine ip aws-notebook
```
    
#### Upload data

(use [docker-machine scp](https://docs.docker.com/machine/reference/scp/))

```
docker-machine scp data/knn_cross_val/xy_file.npz aws-notebook:/home/ubuntu/xy_file.npz
```

#### Run container

Points:
- run the  container interactively so we can copy the secret token
- forward the ipython notebook port, 8888, to the port opened on the instance as a part of `docker-machine`
security group, 8888 (is this necessary since they are the same port?)
- share the docker machine home folder as a volume inside the docker container to allow access to the data and
save notebook

```
docker run -it --rm -p 8888:8888  \
  -v /home/ubuntu:/home/jovyan/work \
  --name aws-py \
  jupyter/scipy-notebook
```

#### Run notebook

To access the notebook server, point browser to: `<docker machine ip>:8888/?token=<token copied from container terminal>`

In `work` folder, create a new notebook and add the following:

```
import os

import numpy as np
from sklearn.model_selection import GridSearchCV
from sklearn.metrics import classification_report
from sklearn.neighbors import KNeighborsClassifier as KNN

# Load data
def load_cross_val_data(datafile):
    npzfile = np.load(datafile)
    X = npzfile['X']
    y = npzfile['y']
    return X,y

datafile = os.path.join('xy_file.npz')
X, y = load_cross_val_data(datafile)

# Tune parameters
tuned_parameters = {'n_neighbors': range(15,95,10)}

clf = GridSearchCV(KNN(n_neighbors=15),
                   tuned_parameters,
                   cv=5,
                   verbose=10)
clf.fit(X, y)

print("Best parameters set found on development set:\n")
print(clf.best_params_)

print("Grid scores on development set:\n")

means = clf.cv_results_['mean_test_score']
stds = clf.cv_results_['std_test_score']
res_params = clf.cv_results_['params']
for mean, std, params in zip(means, stds, res_params):
    print("%0.3f (+/-%0.03f) for %r"
          % (mean, std * 2, params))
```

Running this notebook should result in output:

```
Fitting 5 folds for each of 8 candidates, totalling 40 fits
[CV] n_neighbors=15 ..................................................
...
```

Now go grab a cup of coffee, read a book, start a painting, and let the notebook run and keep producing output. Eventually, it will tell you the best k value for this data and estimator, then it will shut down after 45 minutes of inactivity. When you come back, if it has shut down, the kernel will be disconnected but the output will still be there for you to read.

#### Stop Machine

If you set up the CloudWatch alarm automatically, the machine will be stopped after 45 minutes of inactivity.
To stop the machine manually, in the container terminal, enter `control-C` twice. This will return you to the local terminal. Then type `docker-machine stop aws-notebook`
