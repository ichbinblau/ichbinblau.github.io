---
title: Running the NCSDK examples in Docker
date: 2017-12-19 14:41:32
tags:
- Movidius
- Docker
- NCSDK
---
In Movidius's official documentation, containers are not an explicitly supported platform till Dec, 2017. Hence, I decided to test whether the NCS (neural compute stick) can be accessed from containers.  Benefiting a lot from this [discussion thread](https://ncsforum.movidius.com/discussion/315/linux-virtual-environment-for-ncs), I was able to run the ncsdk examples in Docker container. Here are the steps I took to make it happen. 

## Prerequisites
My test environment:

 - Host: Ubuntu 16.04.2
 - Docker: docker-ce v17.09
 - Movidius compute stick attached

## Build Docker image with ncsdk installed
The [Dockerfile](https://github.com/ichbinblau/ncsdk_container/blob/master/Dockerfile) I used to install the ncsdk and make the examples is borrowed from the instruction. I created a repo and uploaded the Dockerfile and its config files to it. You may check this [branch](https://github.com/ichbinblau/ncsdk_container), and build the Docker image yourself. Run:

```
git clone https://github.com/ichbinblau/ncsdk_container 
cd ncsdk_container
sudo docker build -t the_image_name .
```
**Note**: Do build the image without proxy because ncsdk installer does not take any proxy setting options for the time being. Otherwise, you maybe stuck at installing python dependencies. 

Alternatively, you can download the [image](https://hub.docker.com/r/xshan1/ncsdk_container/) I built on DockerHub straightaway. Run:

```
sudo docker pull xshan1/ncsdk_container
```

## Run ncsdk examples in Docker 
The ncforum provided a way to run Docker container which is accessible to the ncs. Host network mode is recommended to make the usb compute stick visible. 

> There have been some users who have had USB related problems using the Intel NCS within a docker environment and we have found that including the `--net=host` flag can help make the device manager events visible to libusb in a docker environment. 

Therefore, I used the below command:

```
sudo docker run --rm --net=host -it -v /etc/apt/apt.conf:/etc/apt/apt.conf:ro --privileged -v /dev:/dev:shared -v /media/data2/NCS/:/media/data2/NCS/ xshan1/ncsdk_container:latest /bin/bash
```
Then it leads to an interactive terminal in the container like this:

```
movidius@theresa-ubuntu:~/ncsdk$
```
Inside the container, you can run the examples to test the access to ncs.

```
cd examples/apps/hello_ncs_cpp/
make help
make hello_ncs_cpp
make run
```
Then you will check the result by looking at command outputs:

```
movidius@theresa-ubuntu:~/ncsdk/examples/apps/hello_ncs_cpp$ make run

making hello_ncs_cpp
g++ cpp/hello_ncs.cpp -o cpp/hello_ncs_cpp -lmvnc
Created cpp/hello_ncs_cpp executable

making run
cd cpp; ./hello_ncs_cpp; cd ..
Hello NCS! Device opened normally.
Goodbye NCS!  Device Closed normally.
NCS device working.
```
Then you may continue to test the other model with different frameworks such as caffe and tensorflow under `examples` folder. 

## Known issue
When I exit the container and try to create the container again, I got the following error:

```
theresa@theresa-ubuntu:~$ sudo docker run -it --rm --net=host --name=ncsdk -v /etc/apt/apt.conf:/etc/apt/apt.conf:ro --privileged -v /dev:/dev:shared -v /media/data2/NCS/:/media/data2/NCS/ xshan1/ncsdk_container:latest /bin/bash
docker: Error response from daemon: oci runtime error: container_linux.go:265: starting container process caused "process_linux.go:368: container init caused \"open /dev/console: input/output error\"".
```
It looks like that `/dev/console` is locked and not released when container is destroyed. The current workaround I found is to reboot the system to release the lock.  

Overall, this is quick test the possibility which might not be very mature. Welcome your comments and corrections.

---------
References:
1. https://ncsforum.movidius.com/discussion/315/linux-virtual-environment-for-ncs
2. https://github.com/hughdbrown/movidius/tree/master/docker
