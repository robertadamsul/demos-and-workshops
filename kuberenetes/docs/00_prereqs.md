# Kubernetes - Pre-Reqs
K8s - Kubernetes demo project.

These tasks should be performed on __each node__ prior to Kubernetes installation/configuration.

### Add records to the Hosts file
Adding an entry in the hosts file on each node in the Kubernetes cluster ensures that the nodes can always resolve each other's hostnames, without relying on DNS.
```bash 
$ vi /etc/hosts
```
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02  # Worker
```

### Create Firewall rules
We need to open the firewall ports & networks that will be used 
```bash
# ubuntu (or debian)
sudo ufw allow 6443/tcp #apiserver
sudo ufw allow from 10.42.0.0/16 to any #pods
sudo ufw allow from 10.43.0.0/16 to any #services
```
```bash
# RHEL Based (Oracle Linux)
sudo firewall-cmd --permanent --add-port=6443/tcp #apiserver
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16 #pods
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16 #services
sudo firewall-cmd --reload
```

### Update the OS and packages
Its a good idea before installing additional packages and making system config changes that we upgrade our Operating System and currently installed packages:

```bash
# ubuntu (or debian) - Update
$ sudo apt update -y && sudo apt upgrade -y
```
```bash
# RHEL Based (Oracle Linux) - Update
$ dnf update -y
```

### Bash Prompt directory Length (_ubuntu_)
In Ubuntu the default bash prompt will show the full path of your current working directory. *My prefernce is to show only the current directory.* <br/>
To show the current directory, only, in your bash prompt:

```bash
$ export PROMPT_DIRTRIM=1
```

To make this persistent, set the variable in the `~/.bashrc` file:
```bash
# Sets PROMPT_DIRTRIM for every shell login
PROMPT_DIRTRIM=1
```

<br/>


## Install Kubernetes (K3s) - Server / Control-Plane

| __Node:__ | __`demo-k8s-ctr01`__ |
| --- | --- |
| Type:   | Server |
| Roles:  | control-plane, etcd, master |


### Create Configuration File(s):
By default, values present in a YAML file located at `/etc/rancher/k3s/config.yaml` will be used on install 
(this allows overrides on the default K3s config). <br/>
```sh
$ sudo mkdir -p /etc/rancher/k3s
$ sudo vi /etc/rancher/k3s/config.yaml
```

[K3s Server Options](https://docs.k3s.io/cli/server) - create a config file for the server node (Control-Plane). <br/>
On the server node we will be adding a taint, to make this node not available for pods to be scheduled on.
```yaml
tls-san:
  - "demo-k8s-ctr01"
  - "10.61.20.81"
node-taint: 
  - "node-role.kubernetes.io/master=true:NoSchedule"
disable:
  - "traefik"
  - "servicelb"
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-init: true
```

### Install K3s on the Control-Plane:
Install K3s, using the convinience script. (by default this will install the latest "Stable" Release)
```bash
$ curl -sfL https://get.k3s.io | sudo sh -
```
If all went well then you should get an output like:
```
[INFO]  Finding release for channel stable
[INFO]  Using v1.31.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

### Perform initial checks:
Is the cluster running?
```bash
$ sudo kubectl get nodes

...
NAME             STATUS   ROLES                       AGE     VERSION
demo-k8s-ctr01   Ready    control-plane,etcd,master   3m43s   v1.31.3+k3s1
```

Are containers/pods running?
```bash
# Check Pods are running using kubectl
$ sudo kubctl get pods -A

# Check pods/containers are running using crictl
$ sudo crictl ps
```

Check the control-plane node labels:
```bash
$ sudo kubectl get nodes demo-k8s-ctr01 --show-labels

...
NAME             STATUS   ROLES                       AGE   VERSION        LABELS
demo-k8s-ctr01   Ready    control-plane,etcd,master   41m   v1.31.3+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=demo-k8s-ctr01,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

Check the control-plane node taints, this should match what we set in the `/etc/rancher/k3s/config.yaml` config:
```bash
$ sudo kubectl describe node demo-k8s-ctr01 | grep -i 'Taints:'

...
Taints:             node-role.kubernetes.io/master=true:NoSchedule
```

<br/>

### Copy Node Join Token to Agents:
Before we install K3s on the Agents we should copy the join token so this can be called during the Agent install process.
```bash
# copy the node-token to home, and chown
$ sudo cp /var/lib/rancher/k3s/server/node-token ~/k3s-node-token && sudo chown $(whoami):$(whoami) ~/k3s-node-token

# now copy this file over to each node using scp
$ scp ~/k3s-node-token demo-k8s-wrk01:~/k3s-node-token
$ scp ~/k3s-node-token demo-k8s-wrk02:~/k3s-node-token

# finally remove the join token from the home directory
$ rm -f ~/k3s-node-token
```
<br/>

## Install Kubernetes (K3s) - Agent / Workers

| __Node:__ | __`demo-k8s-wrk01` / `demo-k8s-wrk02`__ |
| --- | --- |
| Type:   | Agent/Worker |
| Roles:  | n/a |

The following tasks need to be ran on each of the worker nodes.

Create an SSH session on each to `demo-k8s-wrk01` and `demo-k8s-wrk02`

### Install K3s on the Agent:
_run on the __agent__ nodes_ <br/>
[K3s Agent CLI Options](https://docs.k3s.io/cli/agent) - Additional options are available to be passed over as part of the install. <br/>
To install as an agent/worker node we need to specify the `K3S_URL` to connect to and the `K3S_TOKEN_FILE` we copied over earlier.

```bash
# Elevate to root
$ sudo su

# Install K3s as an Agent, connected to our demo-k8s-ctr01 cluster
$ curl -sfL https://get.k3s.io | K3S_URL=https://demo-k8s-ctr01:6443 K3S_TOKEN_FILE=/tmp/k3s-node-token sh -s -

...
[INFO]  Finding release for channel stable
[INFO]  Using v1.31.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent

# Exit from elevated session
$ exit
```

### Check Nodes in the Cluster:
_run on the __server__ node_ <br/>
Now connect back to your server node `demo-k8s-ctr01` and let check to see if the cluster has the newly added nodes. 
```bash
# Get the nodes, but lets output wide (showing more information).
$ sudo kubectl get nodes -o wide

...
NAME             STATUS   ROLES                       AGE     VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
demo-k8s-ctr01   Ready    control-plane,etcd,master   142m    v1.31.3+k3s1   10.61.20.81   <none>        Ubuntu 24.04.1 LTS   6.8.0-50-generic   containerd://1.7.23-k3s2
demo-k8s-wrk01   Ready    <none>                      13m     v1.31.3+k3s1   10.61.20.82   <none>        Ubuntu 24.04.1 LTS   6.8.0-31-generic   containerd://1.7.23-k3s2
demo-k8s-wrk02   Ready    <none>                      6m52s   v1.31.3+k3s1   10.61.20.83   <none>        Ubuntu 24.04.1 LTS   6.8.0-31-generic   containerd://1.7.23-k3s2
```

### Remove the join token:
_run on the __agent__ nodes_ <br/>
Now that the Agent nodes have joined the cluster we should remove the k3s-node-token we copied over earlier.
```bash
rm /tmp/k3s-node-token
```
<br/>

## Next Step
We now need to install the Kubernetes components so we can create a Cluster.

[__[01_cluster-init]__](/robertadamsul/demos-and-workshops/kubernetes/01_cluster-init.md)
