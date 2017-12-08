
--- 
title: Setup Kubernetes Cluster on Centos 7.3 with Kubeadm
date: 2017-09-20 16:29:53
tags: 
- Kubernetes
- kubeadm
---
The article is to document the steps I took to install Kubernetes cluster on Centos 7.3 with [kubeadm](https://kubernetes.io/docs/admin/kubeadm/). 

## Prerequisites

#### System Configuration

We suppose we have four servers ready, one as k8s master and the other three as k8s nodes. 
Configure local DNS in `/etc/hosts`.  Map the IP address with host names. 
```
192.168.1.102 loadbalancer
192.168.1.101 gateway1
192.168.1.103 gateway2
192.168.1.104 gateway3
```

#### Environment Preparation

We should have at least two servers with Centos 7.3 pre-installed and keep them in the same subnet. 
Optionally, setup proxy if you work behind the corporate network as Kubeadm uses the system proxy to download components. Put the following settings in the `$HOME/.bashrc` file.  Be mindful to put the master host's IP address in the no_proxy list. 
```
export http_proxy=http://proxy_ip:proxy_port/
export https_proxy=http://proxy_ip:proxy_port/
export ftp_proxy=http://proxy_ip:proxy_port/
export no_proxy=192.168.1.102,192.168.1.103,192.168.1.101,192.168.1.104,127.0.0.1,localhost,loadbalancer,gateway1,gateway2,gateway3
```
And check your proxy settings:
```
$ env | grep proxy
```

## Install kubeadm and kubelet on each of your hosts

Add k8s repo to the yum source list.  We suppose you run the command as root. For non-root users, please wrap the command with `sudo bash -c '<command_to_run>'`
```
$ cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
sslverify=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
Install kubeadm and kubelet on each hosts. 
```
$ sudo setenforce 0
$ sudo yum install -y kubelet kubeadm
$ sudo systemctl enable kubelet 
$ sudo systemctl start kubelet
```
By default, the following command installs the latest kubelet and kubeadm. If you want to install a specific version,  you may check the versions. 
```
$ sudo yum list kubeadm  --showduplicates |sort -r
kubeadm.x86_64                        1.7.5-0                        kubernetes
kubeadm.x86_64                        1.7.4-0                        kubernetes
```
And install the specific version ( eg. v1.7.5 ) 
```
$ sudo yum install kubeadm-1.7.5-0
```

## Install Docker on each of your hosts

For the time being, Docker version 1.12.x is still the preferred and verified version that Kubernetes officially supported according to it [doc](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-docker).  But this [thread](https://stackoverflow.com/questions/44657320/which-docker-versions-will-k8s-1-7-support) says that they will add support for Docker 1.13 very soon. 
We will install Docker 1.12 for now. Use the following command to set up the  repository.
```
$ cat << EOF > /etc/yum.repos.d/docker.repo
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
Make yum cache and check possible Docker 1.12.x versions
```
$ sudo yum makecache
$ sudo yum list docker-engine  --showduplicates |sort -r
docker-engine.x86_64             1.12.6-1.el7.centos                 dockerrepo
```
Intall and start Docker service
```
$ sudo yum install docker-engine-1.12.6 -y 
$ sudo systemctl enable docker 
$ sudo systemctl start docker
```
Verify if your Docker cgroup driver matches the kubelet config:
```
$ sudo docker info |grep -i cgroup
$ sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
The default cgroup driver in kubelet config is `systemd`. If Docker's cgroup driver is not `systemd` but `cgroupfs`.  Update the cgroup driver in `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` to  `KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs`
Further, if your Docker runs behind corporate network, set up the proxy in the Docker config:
```
$ cat << EOF >/etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy_ip:proxy_port/" "NO_PROXY=localhost,127.0.0.1,192.168.1.102,192.168.1.103,192.168.1.101,192.168.1.104,loadbalancer,gateway1,gateway2,gateway3"
EOF
```
Then reload the config:
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
$ sudo systemctl restart docker  # restart docker if you add proxy config
```

## Initialize Master

On the master node (load balancer), if you run as root, do
```
$ kubeadm init --apiserver-advertise-address=192.168.1.102 --pod-network-cidr=10.244.0.0/16
```
If you run as a normal user, do
```
$ sudo -E -c "kubeadm init --apiserver-advertise-address=192.168.1.102 --pod-network-cidr=10.244.0.0/16"
```
If `--apiserver-advertise-address` is not specified, it auto-detects the network interface to advertise the master. Better to set the argument if there are more than one network interface.
`--pod-network-cidr`is to specify the virtual IP range for the third party network plugin. We use [flannel](https://github.com/coreos/flannel) as our network plugin here.
Set `--use-kubernetes-version`if you want to use specific Kubernetes version.
To start using your cluster, you need to run (as a regular user):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
By default, your cluster will not schedule pods on the master for security reasons. Expand your load capacity if you want to schedule pods on your master.
```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Install Flannel Pod Network Plugin
A pod network add-on is supposed to be installed in order that pods can communicate with each other. Run:
```
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
If there are more than one NIC, refer to the [flannel issue 39701](https://github.com/kubernetes/kubernetes/issues/39701).
Next, configure Docker with Flannel IP range and settings:
```
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ cat << EOF > /etc/systemd/system/docker.service.d/flannel.conf
[Service]
EnvironmentFile=-/run/flannel/docker
EOF
$ sudo cat << EOF > /run/flannel/docker
DOCKER_OPT_BIP="--bip=10.244.0.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=false"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=10.244.0.1/24 --ip-masq=false --mtu=1450"
EOF
```
Reload the config:
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
Check the status of flannel pods and make sure that it is in `Running` state:
```
$ kubectl get pod --all-namespaces -o wide
```

## Join Nodes to Cluster

Get the cluster token on master:
```
$ sudo kubeadm token list
```
Run the commands below on each of the nodes:
```
$ kubeadm join --token e5e6d6.6710059ca7130394 192.168.1.102:6443
```
Replace `e5e6d6.6710059ca7130394` with the token got from `kubeadm` command.
And check whether nodes joins the cluster successfully.
```
$ kubectl get nodes
```

## Install Dashboard Add-on

Create the dashboard pod :
```
$ kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```
Since Kubernetes v1.6, its API server uses RBAC strategy. The `kubernetes-dashboard.yaml` does not define an valid ServiceAccount. Create file `dashboard-rbac.yaml` and bind account `system:serviceaccount:kube-system:default` with role `ClusterRole cluster-admin`： 
```
$ cat << EOF > dashboard-rbac.yaml 
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin 
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
EOF
```
Define the RBAC rules to the pod and check pod state with `kubectl get po --all-namespace` command after that

```
$ kubectl create -f dashboard-rbac.yaml
```
 
Configure `kubernetes-dashboard` service to use [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):
```
$ cat << EOF > dashboard-svc.yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
EOF
$ kubectl apply -f dashboard-svc.yaml
```
Then get the NodePort. 
```
$ kubectl get svc kubernetes-dashboard -n kube-system
NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   10.102.129.68   <nodes>       80:32202/TCP   12h
```
`32202` in the output is the NodePort. You can visit the dashboard by `http://<master-ip>:<node_port>` now. In our case, the url is `http://192.168.1.102:32202`. 


## Tear Down

Firstly, [drain](https://kubernetes.io/docs/user-guide/kubectl/v1.7/#drain) the nodes on the master or wherever credential is configured. It does a graceful termination and marks the node as unschedulable. 
```
$ kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
$ kubectl delete node <node name>
```
Then on the node to be removed, remove all the configuration files and settings
```
$ sudo kubeadm reset
```

## Diagnose

 - Check services and pods status. `kube-system` is the default namespace for system-level pods. You may also pass other specific namespaces. Use `--all-namespaces` to check all namespaces
 ```
 $ kubectl get po,svc -n kube-system
 ```
 This is how the output looks like:
	```
		NAME                                       READY     STATUS    RESTARTS   AGE
		po/etcd-loadbalancer                       1/1       Running   0          1d
		po/kube-apiserver-loadbalancer             1/1       Running   0          1d
		po/kube-controller-manager-loadbalancer    1/1       Running   0          1d
		po/kube-dns-2425271678-zj91n               3/3       Running   0          1d
		po/kube-flannel-ds-w9dvz                   2/2       Running   0          1d
		po/kube-flannel-ds-zn6c4                   2/2       Running   1          1d
		po/kube-proxy-m6nvj                        1/1       Running   0          1d
		po/kube-proxy-w92kx                        1/1       Running   0          1d
		po/kube-scheduler-loadbalancer             1/1       Running   0          1d
		po/kubernetes-dashboard-3313488171-tkdtz   1/1       Running   0          1d
		
		NAME                       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
		svc/kube-dns               10.96.0.10      <none>        53/UDP,53/TCP   1d
		svc/kubernetes-dashboard   10.102.129.68   <nodes>       80:32202/TCP    1d
	```

 - Check pods logs. Get pod name from the command above (eg. `kubernetes-dashboard-3313488171-tkdtz`). Use `-c <container_name>` if there are more than one containers running in the pod. 
 ```
 $ kubectl logs <pod_name> -f -n kube-system
 ```

 - Run commands in the container. Use `-c <container_name>` if there are more than one containers running in the pod. 
 Run a single command:
 ```
 $ kubectl exec <pod_name> -n <namespace> <command_ to_run>
 ```
 Enter the container's shell:
 ```
 $ kubectl exec -it <pod_name> -n <namespace> -- /bin/bash
 ```

 - Check Docker logs
 ```
 $ sudo journalctl -u docker.service -f 
```

 - Check kubelet logs
 ```
 $ sudo journalctl -u kubelet.service -f 
 ```
 
---------
References:
1. https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
2. https://kubernetes.io/docs/admin/kubeadm/
3. https://github.com/coreos/flannel

