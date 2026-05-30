## Bootstrap a Kubernetes Cluster using Kubeadm

To begin, log in to AWS Console.

### Task 1: Launching Instances on AWS

* `3` instances of `t2.medium` instance with OS version as `Ubuntu 22.04 LTS` in your preferred region.
* `Storage : 10 GB`
* Instead of opening all ports you can open these ports internally.
* `Type : Custom TCP`
* `Source type : Anywhere`
    |      Nodes	      |    Port Number	 |         Use Case                       |
    |---------------------|------------------|----------------------------------------|
    | Master, Workers	  |    `2379-2380`   |  Etcd Client API / Server API          |
    | Master              |       `6443`  	 |  Kubernetes API Server (Secure Port)   |
    | Master, Workers     |   `6782-6784`    |  Weave Net Server/Client API #CNI      |
    | Master, Workers     |   `10250-10255`	 |  Kubelet Communication                 |
    | Workers             |   `30000-32767`	 |  Reserved of NodePort Ips              |	   


* Add the below code in Advanced Details -> User data - optional.
        <details><summary>User Data</summary>
        <p>
        
      #!/bin/bash

set -euxo pipefail

KUBERNETES_VERSION="1.29.0-1.1"
CRIO_VERSION="1.29"

# Disable swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

apt-get update -y

# Kernel modules
cat <<EOF >/etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Sysctl settings
cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sysctl --system

# Install dependencies
apt-get install -y \
curl \
gpg \
apt-transport-https \
ca-certificates \
software-properties-common \
jq

mkdir -p /etc/apt/keyrings

###########################################
# Install CRI-O
###########################################

curl -fsSL \
https://pkgs.k8s.io/addons:/cri-o:/stable:/v${CRIO_VERSION}/deb/Release.key \
| gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo \
"deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] \
https://pkgs.k8s.io/addons:/cri-o:/stable:/v${CRIO_VERSION}/deb/ /" \
> /etc/apt/sources.list.d/cri-o.list

apt-get update -y
apt-get install -y cri-o

systemctl enable crio --now

###########################################
# Install Kubernetes Components
###########################################

curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo \
"deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
> /etc/apt/sources.list.d/kubernetes.list

apt-get update -y

apt-get install -y \
kubelet=${KUBERNETES_VERSION} \
kubeadm=${KUBERNETES_VERSION} \
kubectl=${KUBERNETES_VERSION}

apt-mark hold kubelet kubeadm kubectl

###########################################
# Configure Node IP
###########################################

LOCAL_IP=$(hostname -I | awk '{print $1}')

cat <<EOF >/etc/default/kubelet
KUBELET_EXTRA_ARGS=--node-ip=${LOCAL_IP}
EOF

systemctl restart kubelet

echo "===================================="
echo "Kubernetes Installed Successfully"
echo "CRI-O Installed Successfully"
echo "Node IP: ${LOCAL_IP}"
echo "===================================="

### Task 3: Initializing the Cluster

rename the VM's as Master, Node1 and Node2 from the AWS Console.

Set the hostname to all three nodes as master, Node1, and Node2 in their respective terminals for easy understanding, by running the below command:

Connect to Master.

Switch to root.
```
sudo su
```
```
hostnamectl set-hostname Master
```
```
bash
```

Connect to Node1.

Switch to root.
```
sudo su
```
```
hostnamectl set-hostname Node1
```
```
bash
```

Connect to Node2.

Switch to root.
```
sudo su
```
```
hostnamectl set-hostname Node2
```
```
bash
```

Start kubeadm only on **master**
```
kubeadm init --ignore-preflight-errors=all
```

If the it runs successfully, it will provide a join command which can be used to join the master. Make a note of the highlighted part.
Run the following commands to configure kubectl on master.
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
chown $(id -u):$(id -g) $HOME/.kube/config
```
 
### Task 4: Joining a Cluster

Paste the copied join token on both the worker nodes.

**Note
If you want to list and generate tokens again to join worker nodes, then follow the below steps(optional)
```
kubeadm token list
kubeadm token create  --print-join-command
```

View node information on the **master**
```
kubectl get nodes
```

### Task 5: Deploy Container Networking Interface.
Go to **master** node and 
Apply weave CNI (Container Network Interface) as shown below:
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
(reference link1:- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/)

View nodes to see that they are ready.
```
kubectl get nodes
```

View all Pods including system Pods and see that dns and weave are running.
```
kubectl get pod -n kube-system
```
Note**
Make sure both the worker nodes have one core-dns pod.
If not: Run the following commands
```
kubectl get pods -A -o wide
```
```
kubectl delete pod <corednspodname> -n kube-system
```
Repeat for both pods
Now Verify:
```
kubectl get pods -A -o wide
```


### Task 6: Create Pods

Create a Pod called pod1 based on a Docker image httpd on the master.
```
kubectl run pod1 --image=httpd
```

View the status of Pod and make sure it’s in the running state.
```
kubectl get pods
```

Get a shell to the container in the Pod using the Pod name from the previous step.
```
kubectl exec -it <pod_name> -- /bin/bash
``` 

Install curl in the container
```
apt update
apt install curl -y
```

Run curl on the localhost (container) to verify the http installation.
```
curl localhost 
exit
```



 


