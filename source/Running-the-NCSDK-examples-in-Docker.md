---
title: Running the NCSDK examples in Docker
date: 2017-12-19 14:41:32
tags:
- Movidius
- Docker
- NCSDK
---
In Movidius's official documentation, Docker container is not an explicitly supported platform (as of Dec, 2017). I am working for Intel and I decided to test whether the NCS (Neural Compute Stick) can be accessed from a container. Benefiting from this [discussion thread](https://ncsforum.movidius.com/discussion/315/linux-virtual-environment-for-ncs), I was able to run the ncsdk examples in a Docker container. Here are the steps I took to make it happen. 

## Prerequisites
My test environment:

 - Host: Ubuntu 16.04.2
 - Docker: docker-ce v17.09
 - Movidius compute stick attached

## Build Docker image with ncsdk installed
The [Dockerfile](https://github.com/ichbinblau/ncsdk_container/blob/master/Dockerfile) I used to install the ncsdk and make the examples is based on instructions from [hughdbrown](https://github.com/hughdbrown/movidius/tree/master/docker). I created a repo and uploaded the Dockerfile and its config files to it. You may check this [branch](https://github.com/ichbinblau/ncsdk_container), and build the Docker image yourself. Run:

```
git clone https://github.com/ichbinblau/ncsdk_container 
cd ncsdk_container
sudo docker build -t the_image_name .
```
**Note**: The ncsdk installer currently does not honor any proxy setting options. Build the image without being behind a proxy or the build step may get stuck installing python dependencies from the internet. 

Alternatively, you can download the [image](https://hub.docker.com/r/xshan1/ncsdk_container/) I built on DockerHub straightaway. Run:

```
sudo docker pull xshan1/ncsdk_container
```

## Run ncsdk examples in Docker 
The ncforum provided a way to run a Docker container which is accessible to the NCS. Host network mode is recommended to make the USB compute stick visible. 

> Some users have had USB related problems using the Intel NCS within a Docker environment. We have found that including the `--net=host` flag can help make the device manager events visible to libusb in a Docker environment. 

Therefore, I used this command:

```
sudo docker run --rm --net=host -it -v /etc/apt/apt.conf:/etc/apt/apt.conf:ro --privileged -v /dev:/dev:shared -v /media/data2/NCS/:/media/data2/NCS/ xshan1/ncsdk_container:latest /bin/bash
```
This leads to an interactive terminal in the container looking like this:

```
movidius@theresa-ubuntu:~/ncsdk$
```
Inside the container, you can run the examples to test access to the NCS.

```
cd examples/apps/hello_ncs_cpp/
make help
make hello_ncs_cpp
make run
```
Check the result by looking at the command outputs, such as this:

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
You may continue to test the other model with different frameworks such as [Caffe](http://caffe.berkeleyvision.org/) and [TensorFlow](https://www.tensorflow.org/) under the `examples` folder. 

## Known issue
When I exit the container and try to create the container again, I am getting the following error:

```
theresa@theresa-ubuntu:~$ sudo docker run -it --rm --net=host --name=ncsdk -v /etc/apt/apt.conf:/etc/apt/apt.conf:ro --privileged -v /dev:/dev:shared -v /media/data2/NCS/:/media/data2/NCS/ xshan1/ncsdk_container:latest /bin/bash
docker: Error response from daemon: oci runtime error: container_linux.go:265: starting container process caused "process_linux.go:368: container init caused \"open /dev/console: input/output error\"".
```
It looks like `/dev/console` is locked and not released when the container is destroyed. The workaround I found was to reboot the system to release the lock.  

## Comments welcome
This is quick example of using Docker containers to access the NCS.  I welcome your comments and corrections.

---------
References:
1. https://ncsforum.movidius.com/discussion/315/linux-virtual-environment-for-ncs
2. https://github.com/hughdbrown/movidius/tree/master/docker
