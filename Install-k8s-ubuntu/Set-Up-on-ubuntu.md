14 Steps to Install kubernetes on Ubuntu 20.04(bento/ubuntu-20.04), 18.04(hashicorp/bionic64)

(Note : - This article has been updated for installing kubernetes on ubuntu 20.04)

In this article on How to Install kubernetes on Ubuntu 20.04 and18.04 we are going to setup Kubernetes cluster along with kubeadm on Ubuntu 20.04 (The same steps can be used for the Ubuntu 18.04 as well as 16.04)

We will start setting up everything from the scratch on virtual machine. Since we are setting up kubernetes cluster, so we will be provisioning two virtual machine - kubernetes master(kmaster) and kubernetes kworker(kworker). The only pre-requisite which you need to do is to install Vagrant and Virtual Box before.

We are going to setup two nodes for kubernetes cluster

Master node
Worker node
Prerequisites
Reading time is about 20 minutes
Vagrant 2.2.15 or latest - For installation instruction click here
VM VirtualBox - For installation instruction click here



Step 1 - Start your vagrant box
As a minimum requirement for kubernetes installation we need -

Master Node - 2 cpus, 2 GB Memory
Worker Node - 1 cpu, 1 GB Memory
Use following Vagrantfile or at least create a Vagrantfile and copy the following configuration into it -

Vagrant.configure("2") do |config|
  config.vm.define "master" do |master|
    master.vm.box_download_insecure = true    
    master.vm.box = "bento/ubuntu-20.04"        ## For ubuntu 18.04 use - hashicorp/bionic64
    master.vm.network "private_network", ip: "100.0.0.1"
    master.vm.hostname = "master"
    master.vm.provider "virtualbox" do |v|
      v.name = "master"
      v.memory = 2048
      v.cpus = 2
    end
  end

  config.vm.define "worker" do |worker|
    worker.vm.box_download_insecure = true 
    worker.vm.box = "bento/ubuntu-20.04"        ## For ubuntu 18.04 use - hashicorp/bionic64
    worker.vm.network "private_network", ip: "100.0.0.2"
    worker.vm.hostname = "worker"
    worker.vm.provider "virtualbox" do |v|
      v.name = "worker"
      v.memory = 1024
      v.cpus = 1
    end
  end

end
BASH


Start you virtual boxes by starting up your vagrant box -

vagrant up 
BASH
**Note - If you are interested in setting the kubernetes cluster on Ubuntu 18.04 then replace worker.vm.box = "bento/ubuntu-20.04" with worker.vm.box = "hashicorp/bionic64" **


Step 2 - Update host files on both master and worker node
After starting the vagrant box now we need to login into the virtual machine using the command vagrant ssh master

master node - SSH into the master node

vagrant ssh master
BASH
Add host entry for master as well as worker node

sudo vi /etc/hosts
BASH
100.0.0.1 master.jhooq.com master
100.0.0.2 worker.jhooq.com worker
BASH
worker node - SSH into the master node

vagrant ssh worker
BASH
Add host entry for master as well as worker node

sudo vi /etc/hosts
BASH
100.0.0.1 master.jhooq.com master
100.0.0.2 worker.jhooq.com worker
BASH
Test the worker node by sending from master

ping worker
BASH
PING worker.jhooq.com (100.0.0.2) 56(84) bytes of data.
64 bytes from worker.jhooq.com (100.0.0.2): icmp_seq=1 ttl=64 time=0.462 ms
64 bytes from worker.jhooq.com (100.0.0.2): icmp_seq=2 ttl=64 time=0.686 ms
BASH
Test the master node by sending from worker

ping master
PING master.jhooq.com (100.0.0.1) 56(84) bytes of data.
64 bytes from master.jhooq.com (100.0.0.1): icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from master.jhooq.com (100.0.0.1): icmp_seq=2 ttl=64 time=0.510 ms
BASH


Step 3 - Install Docker on both master and worker node
You need to install Docker on both the node.

So run the following installation command on both the nodes

sudo apt-get update
BASH
sudo apt install docker.io
BASH
Enable and start docker

sudo systemctl enable docker
BASH
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
BASH
sudo systemctl start  docker
BASH
Check the docker service status

sudo systemctl status docker
BASH
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-04-23 19:10:59 UTC; 4s ago
     Docs: https://docs.docker.com
BASH


Step 4 - Disable the firewall and turnoff the "swapping"
We need to disable firewall as well as swapping on master as well as worker node. Because to install kubernetes we need to disable the swapping on both the nodes

sudo ufw disable
BASH
Firewall stopped and disabled on system startup
BASH
sudo swapoff -a
BASH

Step 5 - Install "apt-transport-https" package
To download the kubernetes and its public we need to install "apt-transport-https" package on both master as well as worker node

sudo apt-get update && sudo apt-get install -y apt-transport-https
BASH


Step 6 - Download the public keys
We need to have the public keys for accessing packages on Google Cloud.

So run the following command to get the public keys on both master as well as worker node

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
BASH
OK
BASH

Step 7 - Add kubernetes repo
As a next step we need to add the kubernetes repo to both master as well as worker node

sudo bash -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list'
BASH

Step 8 - Install kubernetes
Now after adding the kubernetes repo we need to install the kubernetes on both mater as well as worker node

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
BASH

Step 9 - Enable and Start kubelet
Alright now we have installed the kubernetes, now we need to enable the kubelet support for both master as well worker node

sudo systemctl enable kubelet
BASH
sudo systemctl start kubelet
BASH

Step 10 - Initialize the kubernetes cluster
Okay now we have reach to point where we have done all the prerequisite for initializing the kubernetes cluster.

Let's run the kubernetes initialization command on only on master

sudo kubeadm init --apiserver-advertise-address=100.0.0.1
BASH
Note down kubeadm join command which we are going to use from worker node to join the master node using token. (Note : - Followig command will be different for you, do not try copy the following command)

sudo kubeadm join 100.0.0.1:6443 --token g2bsw7.5xr3bqc21eqyc6r7 --discovery-token-ca-cert-hash sha256:39b2b0608b9300b3342a8d0a0e9204c8fc74d45b008043a810f94e4f1fb8861f
BASH

Step 11 - Move kube config file to current user (only run on master)
To interact with the kubernetes cluster and to user kubectl command, we need to have the kube config file with us.

Use the following command to get the kube config file and put it under working directory.

mkdir -p $HOME/.kube
BASH
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
BASH
sudo chown $(id -u):$(id -g) $HOME/.kube/config
BASH

Step 12 - Apply CNI from kube-flannel.yml(only run on master)
After the master of the cluster is ready to handle jobs and the services are running, for the purpose of making containers accessible to each other through networking, we need to set up the network for container communication.

Get the CNI(container network interface) configuration from flannel

wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
BASH

Step 13 - Join worker nodes to master(only run on worker)
In the Step 10 we generated the token and kubeadm join command.

Now we need to use that join command from our worker node

sudo kubeadm join 100.0.0.1:6443 --token g2bsw7.5xr3bqc21eqyc6r7     --discovery-token-ca-cert-hash sha256:39b2b0608b9300b3342a8d0a0e9204c8fc74d45b008043a810f94e4f1fb8861f
BASH
W0423 19:27:00.344480   18268 join.go:346] preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
preflight] Running pre-flight checks
	WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
preflight] Reading configuration from the cluster...
preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
kubelet-start] Starting the kubelet
kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
BASH

Step 14 - Check the nodes status(only run on master)
To check the status of the nodes use

kubectl get nodes
BASH
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   26m   v1.18.2
worker   Ready    <none>   63s   v1.18.2
BASH
(Note - In case you see the status of the nodes not ready then I would recommend to check the these troubleshooting step as well as flannel settings)
