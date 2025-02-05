# Kubernetes - Create Pods, Perform Checks and Cleanup
K8s - Kubernetes demo project.

Now have a 3 node kubernetes cluster (1 Server, and 2 Agents).<br/>
The Agents (worker) nodes will run the containers, and the servers role is to manage the Kubernetes API, and control the scheduling of the pods/containers to the appropriate agent.<br/>
We can now have a little play before any "production" workload is released to the cluster.

## Overview
_run on the __server__ node_ <br/>
In this section we will create two webservers, and a debugging pod: 
* web01 - [nginx](https://hub.docker.com/_/nginx/tags) webserver
* web02 - [httpd](https://hub.docker.com/_/httpd) webserver
* debug - [alpine-linux](https://hub.docker.com/_/alpine) (really small linux distro).

```bash
# create pod named web01 using the nginx container image
$ sudo kubectl run web01 --image=nginx

# create pod named web02 using the httpd container image
$ sudo kubectl run web02 --image=httpd
```
OK lets check the status of these pods and check they are running, and make a note of the IP Addresses they've been assigned.
```bash
$ sudo kubectl get pods -o wide

...
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
web01   1/1     Running   0          32s   10.42.3.4   demo-k8s-wrk01   <none>           <none>
web02   1/1     Running   0          26s   10.42.4.6   demo-k8s-wrk02   <none>           <none>
```

Great the pods are running and we have the IP Addresses.<br/>
__note: these addresses are in the CIDR range we defined for the pod-network_ <br/>

## Creating a Debugging Pod:
* **Run interactively & connect:**

    Lets run a pod that we will connect into interactive shell, and we'll set it to remove when we exit.
    ```bash
    # Create the debugging pod, and lets curl the webservers we just created.
    $ sudo kubectl run -i -t --rm debug --image=alpine -- /bin/sh

    ...
    If you don't see a command prompt, try pressing enter.
    / #
    ```
    Now we have a terminal shell session directly into the pod, lets do some debugging of the webservers.
    ```sh
    # Lets ping the pods IP Addresses and see if we get a response
    $ ping 10.42.3.4
    $ ping 10.42.3.4

    # Now lets curl the webserver pods, to do this we need to install curl (as this is a minimal pod).
    $ apk add curl

    # web01 website test
    $ curl -v http://10.42.3.4
    ...
    < Server: nginx/1.27.3
    <title>Welcome to nginx!</title>


    # web02 website test
    $ curl -v http://10.42.4.6
    ...
    < Server: Apache/2.4.62 (Unix)
    <html><body><h1>It works!</h1></body></html>

    # Everything looks good lets exit from the debugging pod
    exit
    ```

    Lets check the pod is removed (or removing)
    ```bash
    $ sudo kubectl get pods

    ...
    NAME    READY   STATUS    RESTARTS   AGE
    web01   1/1     Running   0          19m
    web02   1/1     Running   0          19m
    ```
    <br/>
    <br/>

* **Alternatively: Create a Pod from a manifest file:**

    This time lets create a debugging pod, but this time we'll create it from a manifest file, and we'll add commands so that curl gets installed when the pod starts. <br/>
    *"_Note: The benefit of this approach is, we can store the manifest, and when we want to create a new "Debug" pod, we only need to apply this file."* <br/>
    To keep the pod running indefinately we will get it to run the command `tail -f /dev/null`.

    ```bash
    # Get kubectl to create us a manifest file, that we can use as a boilerplate
    $ sudo kubectl run debug --image=alpine --dry-run=client -o yaml --command -- tail -f /dev/null > debug-pod.yaml
    ```

    Lets take a look at the Pod manifest file that kubectl has created for us.
    ```bash
    $ vi debug-pod.yaml
    ```
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: debug
      name: debug
    spec:
    containers:
    - command:
      - sh
      - -c
      - tail -f /dev/null
      image: alpine
      name: debug
      resources: {}
    dnsPolicy: ClusterFirst
    restartPolicy: Always
    status: {}
    ```

    Lets add additional commands to this manifest container, this will install curl into the Pod for us.
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: debug
      name: debug
    spec:
    containers:
    - command:
      - sh
      - -c
      - apk add --no-cache curl && tail -f /dev/null  # <-- added install curl
      image: alpine
      name: debug
      resources: {}
    dnsPolicy: ClusterFirst
    restartPolicy: Always
    status: {}
    ```
    
    Now we have the manifest file lets apply it using kubectl, wait for the pod to be running, then exec into the pod and check curl is installed.
    ```bash
    # Apply the manifest file
    $ sudo kubectl apply -f debug-pod.yaml

    # check the pod is running.
    $ sudo kubectl get pods debug
    ...
    NAME    READY   STATUS    RESTARTS   AGE
    debug   1/1     Running   0          6m27s

    # Exec into the Debug pod and check curl is installed.
    $ sudo kubectl exec -i -t debug -- /bin/sh
    ...
    / #
    $ curl --help
    Usage: curl [options...] <url>
    -d, --data <data>           HTTP POST data
    -f, --fail                  Fail fast with no output on HTTP errors

    # now exit the pod
    $ exit
    ```

## [Bonus] Create a PowerShell Pod
Bonus Content -- create a debugging Pod that is running PowerShell on linux.
  
We can either install PowerShell ontop of an Linux OS Image (such as Alpine, Ubuntu, ... etc) or we can use a pre-packaged image already available on container registries.

### Create Pod

* **(Option1) Pre-Packaged Image:** <br/>
  This image has already been created by the community it is an Alpine Linux container and PowerShell has been installed onto it.<br/>
  [image:m365pnp/powershell](https://hub.docker.com/r/m365pnp/powershell)

  <details>
    Lets run a pod that we will connect into interactive shell, and we'll set it to remove when we exit.

    ```bash
    # Create the debugging pod, and lets curl the webservers we just created.
    $ sudo kubectl run -i -t --rm powershell --image=m365pnp/powershell:latest --command -- /bin/sh

    ...
    If you don't see a command prompt, try pressing enter.
    / #
    ```

  </details>

<br/>

* **(Option2) Roll your own:** <br/>
  For this we will install powershell ontop of a ubuntu image. Following guide: [Microsoft - Installing PowerShell on Linux](https://learn.microsoft.com/en-us/powershell/scripting/install/install-ubuntu?view=powershell-7.4#installation-via-direct-download)

  <details>
    Lets create a manifest file, and using the command section we will perform the installation steps.

    ```bash
    $ vi powershell-ubuntu.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: powershell-ubuntu
      name: powershell-ubuntu
    spec:
      containers:
      - name: powershell-ubuntu
        image: ubuntu
        command:
        - sh
        - -c
        - |
          apt-get update;
          apt-get install -y wget;
          wget https://github.com/PowerShell/PowerShell/releases/download/v7.4.6/powershell_7.4.6-1.deb_amd64.deb;
          dpkg -i powershell_7.4.6-1.deb_amd64.deb;
          apt-get install -y -f;
          tail -f /dev/null
        readinessProbe:
          exec:
            command: ["test", "-f", "/usr/bin/pwsh"]
          initialDelaySeconds: 5
          periodSeconds: 5
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
    status: {}
    ```

    Connect to the Pod
    ```bash
    # Apply the manifest file
    $ sudo kubectl apply -f powershell-ubuntu.yaml

    # check the pod is running.
    $ sudo kubectl get pods powershell-ubuntu
    ...
    NAME                READY   STATUS    RESTARTS   AGE
    powershell-ubuntu   1/1     Running   0          6m27s

    # Exec into the Debug pod and check curl is installed.
    $ sudo kubectl exec -i -t powershell-ubuntu -- /bin/bash

    ...
    If you don't see a command prompt, try pressing enter.
    / #
    ```
  </details>

### PowerShell(ing)
Now we have a terminal shell session directly into the pod, lets go into powershell.
```bash
# Run PowerShell terminal
$ pwsh
```

```powershell
# Lets run some powershell, cmdlets such as invoke-webrequest
/> Invoke-WebRequest "https://ifconfig.me"
/> $(Invoke-WebRequest "https://ifconfig.me").RawContent
/> Invoke-WebRequest "https://ifconfig.me" | Select-Object RawContent
/> Invoke-WebRequest "https://ifconfig.me" | Select-Object -ExpandProperty RawContent

# Check the content of the root directory using powershell
/> Get-ChildItem /
/> Get-ChildItem | Sort-Object -Descending -Property LastWriteTime | Select-Object Name, Size

# Everything looks good lets exit from powershell, this will take us back to sh
/> exit
```

```sh
# exit the Pod, and let kuberentes delete it for us
$ exit
```

<br/>

## Cleanup
Now we've finished with this exercise lets cleanup, we'll do that by deleting the pods. We can do this by either deleting all pods in this (default) namespace in a single command or by deleting pods by specifying their name.
```bash
# (option 1) Delete all pods in namespace
$ sudo kubectl delete pods --all
...
pod "web01" deleted
pod "web02" deleted
pod "debug" deleted
```
```bash
# (option 2) Delete pods by specifying the pod name, the granular option.
$ sudo kubectl delete pods web01 web02 debug
...
pod "web01" deleted
pod "web02" deleted
pod "debug" deleted
```

## Next Steps
Now we've confirmed we can create pods, and can use them for debugging we should look to create some deployments and access them from outside of the cluster (using an Ingress Controller).
