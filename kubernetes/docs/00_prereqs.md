# Kubernetes - Pre-Reqs
K8s - Kubernetes demo project.

These tasks should be performed on __each node__ prior to Kubernetes installation/configuration.

### Add records to the Hosts file
Adding an entry in the hosts file on each node in the Kubernetes cluster ensures that the nodes can always resolve each other's hostnames, without relying on DNS.
```bash 
$ sudo vi /etc/hosts
```
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01.lan demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01.lan demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02.lan demo-k8s-wrk02  # Worker
```

Make sure each node is able to ping the Control-Plane, and the Control-Plane can ping each Worker.
```bash
ping demo-k8s-ctr01
ping demo-k8s-wrk01
ping demo-k8s-wrk02
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

## Next Step
We now need to install the Kubernetes components so we can create a Cluster.

[__[01_cluster-init]__](/kubernetes/docs/01_cluster-init.md)
