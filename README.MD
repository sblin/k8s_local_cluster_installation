# K8S local cluster installation

This repository describes all the steps to install a local k8s cluster.

All the credits go to [https://github.com/kodekloudhub/certified-kubernetes-administrator-course](https://github.com/kodekloudhub/certified-kubernetes-administrator-course).

I copied the 3 following files:
```
Vagrantfile
    /ubuntu
        update-dns.sh
        /vagrant
            setup-hosts.sh
```

1. Clone

Clone this repository and open a terminal in the directory of the repository.

2.  Start

Create the 3 machines:
```
vagrant up
```

3.  Connect

Connect in SSH to each node:

```
vagrant ssh kubemaster
vagrant ssh kubenode01
vagrant ssh kubenode02
```
4.  Configure

On each node execute the commands:
```
lsmod | grep br_netfilter
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

5.  Install

On each node install kubeadm (as root)
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

6.  Docker

On each node install the cli, containerd and docker

The docker cli:
```
curl https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/docker-ce-cli_20.10.21~3-0~ubuntu-bionic_amd64.deb --output cli.deb
sudo dpkg -i cli.deb
```

Containerd:
```
curl https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/containerd.io_1.6.9-1_amd64.deb --output containerd.deb
sudo dpkg -i containerd.deb
```

Docker:
```
curl https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/docker-ce_20.10.21~3-0~ubuntu-bionic_amd64.deb --output docker.deb
sudo dpkg -i docker.deb 
```

7.  Only on the master node

```
sudo su
rm /etc/containerd/config.toml
systemctl restart containerd
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
```

Then as a regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Keep the output of the previous command. This command will be executed later in each worker node.

8.  Weave

Check that the master node is up and running:
```
kubectl get nodes

NAME         STATUS     ROLES                  AGE     VERSION
kubemaster   NotReady   control-plane,master   6m53s   v1.23.1
```

On the master node, install weave:

```
wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl apply -f weave-daemonset-k8s.yaml
```

9.  Add workers

On each worker node:
```
sudo su
rm /etc/containerd/config.toml
systemctl restart containerd
```

Then execute the command saved in step 7

10. Verify

Check that the worker nodes have joined the cluster
```
kubectl get nodes

NAME         STATUS   ROLES                  AGE   VERSION
kubemaster   Ready    control-plane,master   12m   v1.23.1
kubenode01   Ready    <none>                 51s   v1.23.1
kubenode02   Ready    <none>                 21s   v1.23.1
```

The k8s cluster is up and running.

11. Get access

To get access to the cluster from the host machine, merge the content of the ```.kube/config``` file into your local k8s config file.


