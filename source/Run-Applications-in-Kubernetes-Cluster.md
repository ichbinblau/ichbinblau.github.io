---
title: Run Applications in Kubernetes Cluster
date: 2017-09-21 17:47:11
tags:
- Kubernetes
---

## Prerequisites

We suppose that the Kubernetes cluster is up and running with at least one master and one node.  
Credential has been properly configured and `kubectl` can be used on at least one of the hosts.
Download all the yaml files from [git repo](https://github.com/ichbinblau/SmartHome-Demo.git) and switch to the directory that contains configuration files. 
```
$ git clone https://github.com/ichbinblau/SmartHome-Demo.git
$ git checkout k8s
$ cd SmartHome-Demo/smarthome-web-portal/tools/yamls
```
In order to accelerate the velocity to download the Docker images, we set up a local Docker registry on master host `192.168.1.102`.  Here is the command to start a local registry.  Create a local directory to make the docker images persistent. 
```
$ sudo mkdir -p /var/lib/registry
$ sudo docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /var/lib/registry:/var/lib/registry \
  registry:2
```
You can check the images in the registry by visiting `http://192.168.1.102:5000/v2/_catalog`

## Create a New Namespace

There are a couple of pre-defined namespaces for different purposes (`default`, `kube-system`, `kube-public`). The `namespace.yaml` defines a new namespace named `iot2cloud`. Here is the `namespace.yaml` file:
```
cat namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: iot2cloud
```
We create a new namespace for our applications. 
```
$ kubectl create -f namespace.yaml
```
  You can check the namespaces
```
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
iot2cloud     Active    34s
kube-public   Active    1d
kube-system   Active    1d
```
Voila, we have a new namespace created. In order to simplify the command, we create an alias for the namespace in the context. Alternatively, you can make it persistent by add this to your `$HOME/.bashrc` file. 
```
$ alias kub='kubectl --namespace iot2cloud'
```

## Attach Labels to Nodes

In our scenario, some of the hosts are resource-constrained.  We would like to assign all the Cloud applications to the Master and the others to the Nodes.  
[`nodeSelect`](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector) helps us make this happen. Add labels to categorize the nodes and check the results: 
```
$ kubectl label nodes gateway1 platform=minnowboard
$ kubectl label nodes gateway2 platform=minnowboard
$ kubectl label nodes gateway3 platform=minnowboard
$ kubectl label nodes loadbalancer platform=nuc
$ kubectl get nodes --show-labels
```

## Set up MariaDB

#### Create secrets and configMaps

Kubernetes introduces [secret](https://kubernetes.io/docs/concepts/configuration/secret/#creating-a-secret-using-kubectl-create-secret) concept to hold sensitive information such as password, token, key pairs etc.  We keep our MariaDB password in the `secret` and MariaDB pod will create the password set in the `secret` for user `root `. 
```
$ MYSQL_ROOT_PASS = your_password_to_use
$ echo -n $MYSQL_ROOT_PASS > mysql-root-pass.secret
$ kub create secret generic mysql-root-pass --from-file=mysql-root-pass.secret
```
You can check that the secret was created like this:
```
$ kub get secrets
NAME                  TYPE                                  DATA      AGE
mysql-root-pass       Opaque                                1         2m
$ kub describe secrets/mysql-root-pass
Name:           mysql-root-pass
Namespace:      iot2cloud
Labels:         <none>
Annotations:    <none>

Type:   Opaque

Data
====
mysql-root-pass.secret: 8 bytes
```

#### Create MariaDB Pod and Service 

Remember to update the `image: 192.168.1.102:5000/mariadb:latest` in `mariadb-master-service.yaml` file to the actual repository you use.  **Make the same changes to all the other yaml file.**
```
$ kub create -f mariadb-master-service.yaml # service definition
$ kub create -f mariadb-master.yaml # deployment definition 
```
Check the pods and service status:
```
$ kub get po,svc -o wide
```
Optionally, if you are going to run MariaDB in master-slave mode.  Ensure that there are more than one nodes labeled `nuc` in the cluster and run: 
```
$ kub create -f mariadb-slave-service.yaml
$ kub create -f mariadb-slave.yaml
```

## Run RabbitMQ Service

RabbitMQ service exposes port 5672 catering for requests.
```
$ kub create -f rabbitmq-service.yaml
$ kub create -f rabbitmq.yaml
```

## Create IoT Rest API Service and Gateway Server

There are two containers running in one pod, one for iot-rest-api-service and the other is for gateway server. 
The `replica` is set to 3, meaning 3 pods would be created evenly on `gateway1` to `gateway3` hosts. 
```
$ kub create -f smarthome-gateway-service.yaml
$ kub create -f smarthome-gateway.yaml
```
You can get the `NodePort` of Rest API service:
```
$ kub describe svc/gateway | grep NodePort
Type:                   NodePort
NodePort:               <unset> 32626/TCP
```
And browse the REST service via http://192.168.1.102:32626/api/oic/res

## Run Home Dashboard

2 pods will be created for home dashboard and database will be initialized after running the commands below: 
```
$ kub create -f home-portal-service.yaml
$ kub create -f home-portal.yaml
```
Get the NodePort:
```
$ kub describe  svc/home-portal | grep NodePort
Type:                   NodePort
NodePort:               home    31328/TCP
```
Then you are able to login to the home portal via http://192.168.1.102:31328/ (use the NodePort you got from the `kubectl` command instead)

## Start Admin Portal

Run the following commands to start the admin portal. The admin portal can only run on single pod in that the trained models are stored in the local file system and not yet shared between pods. 
Update the env `http_proxy` and `https_proxy` to empty string in `admin-portal.yaml` if it is not required.  Then run: 
```
$ kub create -f admin-portal-service.yaml
$ kub create -f admin-portal.yaml
```
Get the NodePort and visit the admin portal.  Next, point the `demo` gateway's IP address to the `http://gateway.iot2cloud:8000/`.


## Run Celery Worker and Trigger Tasks

Celery worker is simply a worker process thereby no service definition required.  There are two containers in the pod, one for long running tasks and the other for periodic tasks. 
Run the command below to initialize the worker: 
```
$ kub create -f celery-worker.yaml
```
And run this command  to trigger the tasks (I will try to skip this manual step later):
```
$ kub exec $(kub get po -l app=celery-worker -o name | cut -d'/' -f2) -c long-task -- python CeleryTask/tasks.py
```

## BKMs

**Restart pods**: in some cases, pods get error or fail to restart. I need to restart the pod to recover the application.  There is no straight command to restart pods. You can restart pods by:
```
$ kub delete pods <pod_to_delete>
$ kub create -f <yml_file_describing_pod>
```
If you cannot find the yaml file immediately, run the command below alternatively:
```
$ kub get pod <pod_to_delete> -o yaml | kubectl replace --force -f -
```
**Pod log outputs**
```
$ kub logs -f  <pod_name> -c <container_name>
```

---------
References:
1. http://agiletesting.blogspot.com/2016/11/running-application-using-kubernetes-on.html
2. https://kubernetes.io/docs/concepts/configuration/secret/#overview-of-secrets
3. https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
