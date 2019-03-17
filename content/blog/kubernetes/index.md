---
title: Kubernetes (k8s) local deployment using minikube 
date: '2019-03-11'
---

If you have been involved in building web applications or worked as a DevOps, you must have heard about kubernetes
or k8s (8 replacing the 8 characters between k and s). The fastest way to get up and running with k8s, is to use 
minikube to run k8s locally. If you are keen to learn new technologies, pluralsight is definitely a nice resource
to check. They recently offered access to all their high level content for free for a weekend, and I took advantage
of that and learned the basics of kubernetes.

Kubectl
--------
* The first thing to do when trying to run k8s locally is to install kubectl, the command line tool to manage and
deploy applications on k8s.
* The instructions to do the above are listed in the [docs].
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
* To get a basic understanding of what *apt-key add -* does, see my blog on yarn.
```bash
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
```
* The [tee](https://www.howtoforge.com/linux-tee-command/) command in linux is used to write output to both the standard output and any number of files.
* To check if kubectl was installed, run this in your terminal
```bash
kubectl version --client
```

Minikube
--------
* The instructions for installing minikube are [here](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-minikube):
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube && sudo cp minikube /usr/local/bin && rm minikube
```
* To check if minikube was installed correctly, run
```bash
minikube version
```
* Minikube by default starts a vm, but I run ubuntu inside a vm in windows. minikube does give an option
to specify vmdriver as none using the --vm-driver flag. This runs k8s components on the host, and not the VM.

Ugly Part
---------
* Starting minikube, did not work the way I expected it to
```bash
minikube start --vm-driver=none
```
Resulting in this output
```bash
💣  Failed to update cluster: downloading binaries: copy: error creating file at /usr/bin/kubeadm: open /usr/bin/kubeadm: permission denied
```
Passing most of the the environment variables, and running as root (sudo -E, the E flag passes the user 
environment variables to root), made it go a bit further, but minikube crashed again
```bash
Waiting for pods: apiserver proxy💣  Error restarting cluster: wait: waiting for k8s-app=kube-proxy: timed out waiting for the condition
```

The fix (Let there be light)
-------
* Downgrading minikube to version 0.34 did the trick for me. The solution was posted [here](https://github.com/kubernetes/minikube/issues/3844#issuecomment-472397540)
* The following commands fixed it!
```bash
curl -LO https://storage.googleapis.com/minikube/releases/v0.34.1/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube
CHANGE_MINIKUBE_NONE_USER=true sudo -E minikube start --vm-driver none 
``` 
* The first bash command, downloads the 0.34.1 version of minikube, and installs it in /usr/local/bin. Few notes abt the flags used with curl
    * L flag makes curl redo it's action if the server sends a 3XX response
    * O flag keeps the name of the local (downloaded) file and remote file same.
* In the second command CHANGE\_MINIKUBE\_NONE\_USER=true should move the config files to the correct destination, and adjust the
    permissions. However, I think, we need to define more environment variables to make it work properly. 
    More context [here](https://github.com/kubernetes/minikube/issues/2176#issuecomment-343250011)
   