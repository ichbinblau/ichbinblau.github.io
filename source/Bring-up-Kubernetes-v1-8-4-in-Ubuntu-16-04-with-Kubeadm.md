---
title: Bring up Kubernetes v1.8.4 on Ubuntu 16.04 LTS with kubeadm
date: 2017-12-06 17:20:04
tags:
- Kubernetes
- kubeadm
---
Kubernetes has released v1.8 since Sepetember 2017. The former installation steps for v1.75 are not compatible to the new version. The article is to document the steps I took to install Kubernetes cluster on Ubuntu Server 16.04 LTS with [kubeadm](https://kubernetes.io/docs/admin/kubeadm/). The steps are tested by installing Kubernetes v1.8.4.   

## Prerequisites
Install Ubuntu Server 16.04 LTS using the HWE kernel (version 4.10) option. The HWE kernel is version 4.10 and aims to support newer platforms, while the Ubuntu Sever 16.04 LTS standard kernel is version 4.40. 

#### Environment Preparation

Setup proxy if you work behind the corporate network as Kubeadm uses the system proxy to download components. Put the following settings in the `$HOME/.bashrc` file.  Be mindful to put the master host's IP address in the no_proxy list.
```
export http_proxy=http://proxy_ip:proxy_port/
export https_proxy=http://proxy_ip:proxy_port/
export ftp_proxy=http://proxy_ip:proxy_port/
export no_proxy=192.168.1.102,192.168.1.103,192.168.1.101,192.168.1.104,127.0.0.1,localhost,loadbalancer,gateway1,gateway2,gateway3
```
And check your proxy settings:
```
env | grep proxy
```
Next perform a software update for all packages before you continue with the cluster installation.

#### Install latest OS updates
```
sudo apt update && sudo apt upgrade
```

#### Disable Swap
Since v1.8, it is required to turn swap off. Otherwise, kubelet service cannot be started. We disable system swap by run this command:
```
sudo swapoff -a
```

#### System Configuration

We suppose that we have four servers ready, one as k8s master and the other three as k8s nodes.
Configure local DNS in `/etc/hosts`.  Map the IP address with host names.
```
192.168.1.102 loadbalancer
192.168.1.101 gateway1
192.168.1.103 gateway2
192.168.1.104 gateway3
```

## Install Kubernetes version 1.8.4 on each of your hosts.
```
sudo apt install -y kubeadm kubelet kubectl
```

## Install Docker version 17.09 on each of your hosts
```
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable"
sudo apt-get update && sudo apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.09 | head -1 | awk '{print $3}')
sudo systemctl enable docker 
```
And ensure that the service is up and running:
```
sudo systemctl status docker
```
**Note**: Make sure that the cgroup driver used by kubelet is the same as the one used by Docker. To ensure compatibility you can either update Docker settings (like what the [official document](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-docker) recommends) or update kubelet setting by adding the setting option below to  file `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`:
```
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
```
After updating the service file (for either kubelet or docker service), do remember to reload the configuration and restart the service
```
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl restart kubelet
```
Other than that, it is highly recommended that you use the `overlay2` driver which is faster and stronger than other docker storage drivers. You may follow the instructions [here](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/#prerequisites) to complete the installation. 

## Initialize Kubernetes Master

On the master node (load balancer), if you run as root, do
```
kubeadm init --apiserver-advertise-address=192.168.1.102 --pod-network-cidr=10.244.0.0/16 --skip-preflight-checks
```
If you run as a normal user, do
```
sudo -E kubeadm init --apiserver-advertise-address=192.168.1.102 --pod-network-cidr=10.244.0.0/16 --skip-preflight-checks
```
If `--apiserver-advertise-address` is not specified, it auto-detects the network interface to advertise the master. Better to set the argument if there are more than one network interface.
`--pod-network-cidr`is to specify the virtual IP range for the third party network plugin.
Set `--use-kubernetes-version`if you want to use a specific Kubernetes version.
To start using your cluster, you need to run (as a regular user):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
By default, your cluster will not schedule pods on the master for security reasons. Expand your load capacity if you want to schedule pods on your master.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Install WeaveNet Pod Network Plugin
A pod network add-on is supposed to be installed in order that pods can communicate with each other. 
Set `/proc/sys/net/bridge/bridge-nf-call-iptables` to 1 by adding `net.bridge.bridge-nf-call-iptables=1` to file `/etc/sysctl.d/k8s.conf` in order to pass bridged IPv4 traffic to iptables' chains. 
And run the following command to make it take effect:
```
sudo sysctl -p /etc/sysctl.d/k8s.conf
```
Then run:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Check the status of Weave pods and make sure that it is in `Running` state:
```
kubectl get pod --all-namespaces -o wide
```

## Join Nodes to Cluster

Get the cluster token on master:
```
sudo kubeadm token list
```
Since Kubernetes v1.8, the token is only valid for 24 hours. You may generate another token if the previous one gets expired. 
```
sudo kubeadm token generate
```
Run the commands below on each of the nodes:
```
sudo kubeadm join --token 15491e.0c0c9c99dfbbe690 192.168.1.102:6443 --discovery-token-ca-cert-hash sha256:82a08ef9c830f240e588a26a8ff0a311e6fe3127c1ee4c5fc019f1369007c0b7 --skip-preflight-checks 
```
Replace the token `e5e6d6.6710059ca7130394` and the sha256 hash with the actual token and hash got from `kubeadm init` command.
"Pub key validation" can be skipped passing `--discovery-token-unsafe-skip-ca-verification flag` instead of using `--discovery-token-ca-cert-hash` but the security would be weakened; 
And check whether nodes joins the cluster successfully.
```
kubectl get nodes
```

## Install Dashboard Add-on

Create the dashboard pod :
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
To start using Dashboard run following command:
```
kubectl proxy --address="<ip-addr-listen-on>" -p <listening-port>
eg. $kubectl proxy –-address="192.168.1.102" –p 8001
```
Then access the dashboard at http://192.168.1.102:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/. Replace the IP address with the actually IP you are using. 
There are a couple of ways to login [here](https://github.com/kubernetes/dashboard/wiki/Access-control#authentication). For development purpose, you may simply grant full admin privileges to Dashboard's Service Account by creating below ClusterRoleBinding. Copy the contents below and save as a file named `dashboard-admin.yaml`.  Use `kubectl create -f dashboard-admin.yaml` to deploy it. Afterwards you can use Skip option on login page to access Dashboard.

## Tear Down

Firstly, [drain](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#tear-down) the nodes on the master or wherever credential is configured. It does a graceful termination and marks the node as unschedulable.
```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```
Then on the node to be removed, remove all the configuration files and settings
```
sudo kubeadm reset
```

## Diagnose

 - Check services and pods status. `kube-system` is the default namespace for system-level pods. You may also pass other specific namespaces. Use `--all-namespaces` to check all namespaces
 ```
 kubectl get po,svc -n kube-system
 ```
 This is how the output looks like:
	```
		NAME                                       READY     STATUS    RESTARTS   AGE
		po/etcd-loadbalancer                       1/1       Running   0          9m
		po/kube-apiserver-loadbalancer             1/1       Running   0          9m
		po/kube-controller-manager-loadbalancer    1/1       Running   0          10m
		po/kube-dns-545bc4bfd4-2qvkk               3/3       Running   0          10m
		po/kube-proxy-6rk26                        1/1       Running   0          10m
		po/kube-proxy-qvhmw                        1/1       Running   0          1m
		po/kube-scheduler-loadbalancer             1/1       Running   0          9m
		po/kubernetes-dashboard-7486b894c6-dw8zz   1/1       Running   0          23s
		po/weave-net-s59fw                         2/2       Running   0          3m
		po/weave-net-zsfls                         2/2       Running   1          1m
		 
		NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
		svc/kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   10m
		svc/kubernetes-dashboard   ClusterIP   10.110.76.10   <none>        443/TCP         23s
	```

 - Check pods logs. Get pod name from the command above (eg. `kubernetes-dashboard-3313488171-tkdtz`). Use `-c <container_name>` if there are more than one containers running in the pod.
 ```
 kubectl logs <pod_name> -f -n kube-system
 ```

 - Run commands in the container. Use `-c <container_name>` if there are more than one containers running in the pod.
 Run a single command:
 ```
 kubectl exec <pod_name> -n <namespace> <command_ to_run>
 ```
 Enter the container's shell:
 ```
 kubectl exec -it <pod_name> -n <namespace> -- /bin/bash
 ```

 - Check Docker logs
 ```
 sudo journalctl -u docker.service -f
 ```

 - Check kubelet logs
 ```
 sudo journalctl -u kubelet.service -f
 ```

---------
References:
1. https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
2. https://kubernetes.io/docs/admin/kubeadm/
