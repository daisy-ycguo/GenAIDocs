# Kubernetes installation using kubespray

In this document, we'll install Kubernetes v1.29 using [Kubespray](https://github.com/kubernetes-sigs/kubespray) on a 2 node cluster. 

There are several ways to use Kubespray to deploy a Kubernetes cluster. In this document, we choose to use the Ansible way. For other ways to use Kubespary, refer to [Kubespray's document](https://github.com/kubernetes-sigs/kubespray).

## Node preparation

| hostname   | ip address         | Operating System |
| ---------- | ------------------ | ---------------- |
| k8s-master | 192.168.121.35/24  | Ubuntu 22.04     |
| k8s-worker | 192.168.121.133/24 | Ubuntu 22.04     |

 We assume these two machines are used for Kubernetes 2-node cluster. They have direct internet access both in bash terminal and in apt repository.

 If on any of the above 2 nodes, you have previously installed either Kubernetes, or any other container runtime(i.e. docker, containerd, etc.), please make sure you have clean-up those first. Refer to [Kubernetes installation demo using kubeadm](./k8s_install_kubeadm.md) to clean up the environment.

## Prerequisites

 We assume that there is a third machine as your operating machine. You can log in to this machine and execute the ansible command. Any of the above two K8s nodes can be used as the operating machine. Unless otherwise specified, all the following operations are performed on the operating machine.

Please make sure that the operating machine can login to both K8s nodes via SSH without a password prompt. There are different ways to configure the ssh login without password promotion. A simple way is to copy the public key of the operating machine to the K8s nodes. For example:

```
# generate key pair in the operation machine
$ ssh-keygen -t rsa -b 4096
# manually copy the public key to the K8s master and worker nodes
$ cat ~/.ssh/id_rsa.pub | ssh username@k8s-master "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
$ cat ~/.ssh/id_rsa.pub | ssh username@k8s-worker "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

```

## Step 1. Set up Kubespray and Ansible

Python3 is required in this step. If you don't have, go to [Python website](https://docs.python.org/3/using/index.html) for installation guide.

You shall set up a Python virtual environment and install ansible and other Kubespray dependencies. Simply, you can just run following commands. You can also go to [kubespray ansible installation guide](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible/ansible.md#installing-ansible) for details. 

```
git clone https://github.com/kubernetes-sigs/kubespray.git
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip install -U -r requirements.txt
```

## Step 2. Build your own inventory

Ansible inventory defines the hosts and groups of hosts on which Ansible tasks are to be executed. You can copy a sample inventory with following command:

```
cp -r inventory/sample inventory/mycluster
```

Edit your inventory file `inventory/mycluster/inventory.ini` to config the node name and IP address. The inventory file used in ths demo is as following:
```
[all]
k8s-master ansible_host=192.168.121.35
k8s-worker ansible_host=192.168.121.133

[kube_control_plane]
k8s-master

[etcd]
k8s-master

[kube_node]
k8s-master
k8s-worker

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

## Step 3. Deploy Kubernetes

You can clean up old Kubernetes cluster with Ansible Playbook with following command:
```
# Clean up old Kubernetes cluster with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example cleaning up SSL keys in /etc/,
# uninstalling old packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
# And be mind it will remove the current kubernetes cluster (if it's running)!
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root reset.yml
```

Then you can deploy kubernetes with Ansible Playbook with following command:

```
# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

The Ansible Playbooks will take several minutes to finish.

## Step 4. Create kubectl configuration

On node <b>k8s-master</b>, run the following commands:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you want to access this Kubernetes cluster from other machines, you can install kubectl by `sudo apt-get install -y kubectl` and make the similiar configuration as in node <b>k8s-master</b>.

Then run following command to check the status of your Kubernetes cluster:
```
$ kubectl get node
NAME          STATUS   ROLES           AGE     VERSION
k8s-master    Ready    control-plane   31m     v1.29.5
k8s-worker   Ready    <none>          7m31s   v1.29.5
$ kubectl get pods -A
NAMESPACE                    NAME                                       READY   STATUS    RESTARTS   AGE
kube-system                  calico-kube-controllers-68485cbf9c-vwqqj   1/1     Running   0          23m
kube-system                  calico-node-fxr6v                          1/1     Running   0          24m
kube-system                  calico-node-v95sp                          1/1     Running   0          23m
kube-system                  coredns-69db55dd76-ctld7                   1/1     Running   0          23m
kube-system                  coredns-69db55dd76-ztwfg                   1/1     Running   0          23m
kube-system                  dns-autoscaler-6d5984c657-xbwtc            1/1     Running   0          23m
kube-system                  kube-apiserver-satg-opea-0                 1/1     Running   0          24m
kube-system                  kube-controller-manager-satg-opea-0        1/1     Running   0          24m
kube-system                  kube-proxy-8zmhk                           1/1     Running   0          23m
kube-system                  kube-proxy-hbq78                           1/1     Running   0          23m
kube-system                  kube-scheduler-satg-opea-0                 1/1     Running   0          24m
kube-system                  nginx-proxy-satg-opea-3                    1/1     Running   0          23m
kube-system                  nodelocaldns-kbcnv                         1/1     Running   0          23m
kube-system                  nodelocaldns-wvktt                         1/1     Running   0          24m
```
Now congratulations. Your two-node K8s cluster is ready to use.

## Quick reference

### How to deploy a single node Kubernetes?

Deploying single-node K8s is similiar as two-node K8s cluster. Follow the previous `Step 1. Set up Kubespray and Ansible` to set up environment. 

And then in `Step 2. Build your own inventory`, you can create single-node Ansible inventory by copying the single-node inventory sample as following:

```
cp -r inventory/local inventory/mycluster
```

Edit your single-node inventory to replace the node name from `node1` to your real node name for example `k8s-master` using following command:

```
sed -i "s/node1/k8s-master/g" inventory/mycluster/hosts.ini
```

Then your single-node inventory will look like below:

```
k8s-master ansible_connection=local local_release_dir={{ansible_env.HOME}}/releases

[kube_control_plane]
k8s-master

[etcd]
k8s-master

[kube_node]
k8s-master

[k8s_cluster:children]
kube_node
kube_control_plane
```

And then follow the previous `Step 3. Deploy Kubernetes` and `Step 4. Create kubectl configuration`, you will get a 1-node K8s ready.

### How to scale Kubernetes cluster to add more nodes?

Assume you've already have a two-node K8s cluster and you want to scale it to three nodes. The third node information is:

| hostname   | ip address         | Operating System |
| ---------- | ------------------ | ---------------- |
| third-node | 192.168.121.134/24  | Ubuntu 22.04     |

Make sure the third node has internet access and can be logged in via `SSH` without password promotion from your operating machine.

Edit your Ansible inventory file to add the third node information to `[all]` and `[kube_node]` section as following:
```
[all]
k8s-master ansible_host=192.168.121.35
k8s-worker ansible_host=192.168.121.133
third-node ansible_host=192.168.121.134

[kube_control_plane]
k8s-master

[etcd]
k8s-master

[kube_node]
k8s-master
k8s-worker
third-node

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

Then you can deploy kubernetes to the third node with Ansible Playbook with following command:

```
# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
ansible-playbook -i inventory/mycluster/inventory.ini --limit third-node --become --become-user=root scale.yml -b -v
```
When the playbook is excuted successfully, you can check if the third node is ready with following command:
```
kubectl get nodes
```

For more information, you can visit [Kubespray document](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/nodes.md#addingreplacing-a-worker-node) for adding/removing Kubernetes node.

### How to config proxy?

If your nodes need proxy to access internet, you will need extra configurations during deploying K8s. We assume your proxy is as below:
```
- http_proxy="http://proxy.fake-proxy.com:911"
- https_proxy="http://proxy.fake-proxy.com:912"
```

