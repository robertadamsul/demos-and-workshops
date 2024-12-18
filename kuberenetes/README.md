# demo-kubernetes
K8s - Kubernetes demo project.

## Overview
### Kubernetes Distro:<br/>
[K3s Docs](https://docs.k3s.io/) - K3s is a lightweight, easy-to-install Kubernetes distribution that is ideal for a 3-node cluster because it simplifies deployment and management, while providing the full functionality of Kubernetes with a smaller resource footprint.

### Nodes:
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02  # Worker
```
### Networks:
```bash
# Network Ports and CIDRs
6443/tcp        # Kubernetes apiserver port (used by kubectl)
10.42.0.0/16    # Pod Network used by Kubernetes
10.43.0.0/16    # Service Network used byKubernetes
```
<br/>
