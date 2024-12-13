# demo-kubernetes
K8s - Kubernetes demo project.


## Nodes
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02  # Worker
```

## Prep Config (optional)
These tasks should be performed on each node prior to Kubernetes installation/configuration.


### Bash Prompt directory Length
In Ubuntu the default bash prompt will show the full path of your current working directory.
<br/>*My prefernce is to show only the current directory* 
<br/>
<br/>To show the current directory in your bash prompt:

```bash
$ export PROMPT_DIRTRIM=1
```

To make this persistent, set the variable in the `~/.bashrc` file:
```bash
# Sets PROMPT_DIRTRIM for every shell login
PROMPT_DIRTRIM=1
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

<br/>


## Intstall Kubernetes (K3s)
[K3s Docs](https://docs.k3s.io/) - K3s is a lightweight, easy-to-install Kubernetes distribution that is ideal for a 3-node cluster because it simplifies deployment and management, while providing the full functionality of Kubernetes with a smaller resource footprint.

### Create Configuration File(s)
By default, values present in a YAML file located at `/etc/rancher/k3s/config.yaml` will be used on install. <br/>
This allows us specify configs to apply to the cluster as part of our install.

```sh
$ sudo mkdir -p /etc/rancher/k3s
```

### Control-Plane [demo-k8s-ctr01] :

#### Create Configuration File(s)
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

#### Install K3s on the Control-Plane
Install K3s, using the convinience script. (by default this will install the latest "Stable" Release)
```bash
curl -sfL https://get.k3s.io | sudo sh -
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

#### Perform initial checks:
Is the cluster running?
```bash
$ sudo kubectl get nodes
```
outputs:
```
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
