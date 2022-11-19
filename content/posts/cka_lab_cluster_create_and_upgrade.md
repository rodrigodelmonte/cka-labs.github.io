+++
title = "CKA Lab - Cluster Create & Upgrade"
date = "2022-11-19T16:03:43+01:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
# CKA Lab - Cluster Create & Upgrade


## Objective

By the end of this tutorial, you will:
1. Use `kubeadm` to install a basic cluster
2. Perform a version upgrade on a Kubernetes cluster using `kubeadm`

## Pre-requisites

Git clone the repository code.
```sh
git clone 
```

- The required binaries will be installed by the `./provision.sh`  script when `vagrant up` command runs.
- The tutorial requires [Vagrant](https://developer.hashicorp.com/vagrant/docs/installation) and [Virtualbox](https://www.virtualbox.org/wiki/Downloads) already installed in your local environment.

## Create the cluster

### Control Plane

Launch the empty nodes and connect to the `control01` and run.
```sh
vagrant up
vagrant ssh control01
```

Create the `control-plane` node.
```sh
sudo kubeadm init \
--apiserver-advertise-address=192.168.56.50 \
--pod-network-cidr=10.244.0.0/16 
```

*Flannel requires the range `10.244.0.0/16`, otherwise, the default flannel configs should be updated.* 

Create the `kubeconfig` manifest.
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install the network CNI, otherwise, the control-plane does not get ready.
```sh
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Check if the `control-plane` is ready.
```sh
kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
control01     Ready    control-plane   64s   v1.24.7
```

### Worker

Connect to the `node01` and run `kubeadm join`  command.
```sh
vagrant ssh node01

# THE COMMAND BELLOW IS JUST AN EXAMPLE. YOU CAN GET THIS COMMAND FROM THE `kubeadm init` OUTPUT COMMAND WHEN THE CONTROL-PLANE WAS CREATED.
sudo kubeadm join 192.168.56.50:6443 \
--token 9tjntl.1XXXXXXXXXXX8a0ui \
--discovery-token-ca-cert-hash sha256:381165c9a9f19a123bd0feXXXXXXXXXXXXXXXXXXXXXXXXXX86b92b

# IF IT IS NOT POSSIBLE, REGENERATE THE `kubeadm join` COMMAND. RUN INSIDE THE CONTROL-PLANE.
kubeadm token create --print-join-command
```

*Run the same steps for the `node02`*

Connect into the `control-plane` and check if the `worker` nodes joined the cluster.
```sh
$ kubectl get nodes
NAME            STATUS   ROLES           AGE   VERSION
control01       Ready    control-plane   11h   v1.24.7
worker-node01   Ready    <none>          11h   v1.24.7
worker-node02   Ready    <none>          11h   v1.24.7
```
*Remember to create the Kubeconfig manifest into the the worker nodes, you can use kubeconfig file from the control-plane as an example*

## Upgrade the cluster

### Control Plane

Connect to the `control01`
```sh
vagrant ssh control01
```

Update the `kubeadm` to the new version.
```sh
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.25.1-00 && \
sudo apt-mark hold kubeadm
```

```sh
sudo kubeadm version # Check if the binary is updated
sudo kubeadm upgrade plan # Describe the upgrade plan versions
```

Update the `control-plane` to the new version.
```sh
# Run the upgrade just in one control-plane node.
sudo kubeadm upgrade apply v1.25.1 # New version
```
*If the cluster has HA more than 1 control plane node, run the command below to upgrade the other control plane nodes, instead of  `kubeadmin upgrade apply`*.
```sh
# YOU CAN SKIP THIS STEP IF THE CLUSTER HAS JUST ONE NODE.
sudo kubeadm upgrade node 
```

Update the `kubelet` and `kubectl` to new version.
```sh
kubectl drain control01 --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=1.25.1-00 kubectl=1.25.1-00 && \
sudo apt-mark hold kubelet kubectl

# Restart Kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Bring the node back
kubectl uncordon control01
```

Check if the `control-plane` is running with the new version.
```sh
sudo kubectl get nodes
NAME            STATUS   ROLES           AGE   VERSION
control01       Ready    control-plane   15m   v1.25.1
worker-node01   Ready    <none>          12m   v1.24.7
worker-node02   Ready    <none>          11m   v1.24.7
```

### Worker

Connect to the `node01` and update the `kubeadm` to the new version.
```sh
vagrant ssh node01
```

Update the `kubeadm` to the new version.
```sh
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.25.1-00 && \
sudo apt-mark hold kubeadm
```

Update the `worker` to the new version.
```sh
sudo kubeadm upgrade node
```

The worker node needs the `kubeconfig` credentials to allow the communication between the `kubectl`  client and the  `control-plane`. For this reason, copy the `kubeconfig`  created during the cluster creation to the workers.
```sh
# RUN INSIDE THE CONTROL-PLANE NODE
cp ~/.kube/config /vagrant/kubeconfig 

# RUN INSIDE THE WORKER NDOES
mkdir ~/.kube
cp /vagrant/kubeconfig ~/.kube/config
```

Update the `kubelet` and `kubectl` to new version.
```sh
kubectl drain worker-node01 --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=1.25.1-00 kubectl=1.25.1-00 && \
sudo apt-mark hold kubelet kubectl

# Restart Kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Make the node back
kubectl uncordon worker-node01
```

*Repeat the Upgrade Worker steps again in the `node02`, remember to update the node name to `worker-node02`  and `node02` when required.*

Check if all the nodes is running the new Kubernetes version!
```sh
kubectl get nodes
NAME            STATUS   ROLES           AGE   VERSION
control01       Ready    control-plane   41m   v1.25.1
worker-node01   Ready    <none>          37m   v1.25.1
worker-node02   Ready    <none>          36m   v1.25.1
```

**Congrats, you finished this tutorial!!!***

#### References
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- https://github.com/flannel-io/flannel#deploying-flannel-manually
- https://github.com/David-VTUK/CKA-StudyGuide
- https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.25.pdf
