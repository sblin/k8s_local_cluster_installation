# K8S local cluster installation

Modification history:

- September 1, 2023:
    - Update point 5 to get newer versions of docker
    - Update point 6 with k8s version 1.28   
***
This repository describes all the steps to install a local k8s cluster using Docker and Virtual Box.

Credits for `Vagrantfile`, `update-dns.sh` and `setup-hosts.sh` go to [https://github.com/kodekloudhub/certified-kubernetes-administrator-course](https://github.com/kodekloudhub/certified-kubernetes-administrator-course).


## 1. Clone

Clone this repository and open a terminal in the directory of the repository.   
Change the Vagranfile to adapt the number of hosts and memory settings (default is 1 master node, 1 runner node, 2048M/node)

## 2.  Create the machines

```
vagrant up
```

## 3.  Connect to each machine

```
vagrant ssh kubemaster
vagrant ssh kubenode01
```

In the following chapter we follow instructions described in https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


## 4.  Configure

On each node check for the `br_netfilter` module. If not loaded, load the module:
```
lsmod | grep br_netfilter
sudo modprobe br_netfilter
```
Execute the script:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

## 5.  Install docker 

On each node install the cli, containerd and docker (we can change for newer versions)

The docker cli:
```
curl https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/docker-ce-cli_24.0.5-1~ubuntu.20.04~focal_amd64.deb --output cli.deb
sudo dpkg -i cli.deb
```

Containerd:
```
curl https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/containerd.io_1.6.9-1_amd64.deb --output containerd.deb
sudo dpkg -i containerd.deb
```

Docker:
```
curl https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/docker-ce_24.0.5-1~ubuntu.20.04~focal_amd64.deb --output docker.deb
sudo dpkg -i docker.deb 
```

## 6. Install kubelet, kubeadm and kubectl

On each node install kubeadm:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

On Ubuntu 20.04 `/etc/apt/keyrings` does not exist:
```
sudo mkdir -755 /etc/apt/keyrings
```

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```



Add the Kubernetes apt repository:
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo snap install kubelet --classic
sudo snap install kubeadm --classic
sudo snap install kubectl --classic
sudo apt-mark hold kubelet kubeadm kubectl
```

## 7.  Create the cluster

**** Only on the master node ****

```
sudo su
rm /etc/containerd/config.toml
systemctl restart containerd
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
```
Keep the output of the command. This command will be executed later in each worker node.


Then as a regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 8.  Weave

Weave (https://www.weave.works/docs/net/latest/overview/) is used to define a virtual network between the docker containers.

Check that the master node is up and running:
```
kubectl get nodes

NAME         STATUS     ROLES                  AGE     VERSION
kubemaster   NotReady   control-plane,master   6m53s   v1.23.1
```

On the master node, get weave:

```
wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

Modify the definition to avoid the use of a Network Policy Controller (in order to allow access to the internet from the pods) and then apply the definition.  

Set `EXPECT_NPC` to "0"

```yaml
containers:
    - name: weave
        env:
        - name: INIT_CONTAINER
            value: "true"
        - name: EXPECT_NPC
            value: "0"
```

```
kubectl apply -f weave-daemonset-k8s.yaml
```

## 9.  Add workers

On each worker node:
```
sudo su
rm /etc/containerd/config.toml
systemctl restart containerd
```

Then execute the command saved in step 7

## 10. Verify

Check that the worker nodes have joined the cluster
```
kubectl get nodes

NAME         STATUS   ROLES                  AGE   VERSION
kubemaster   Ready    control-plane,master   12m   v1.23.1
kubenode01   Ready    <none>                 51s   v1.23.1
```

The k8s cluster is up and running.

## 11. Get access

To get access to the cluster from the host machine, merge the content of the `.kube/config` file into your local k8s config file.


