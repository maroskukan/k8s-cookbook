# Kubernetes

- [Kubernetes](#kubernetes)
  - [Documentation](#documentation)
  - [Essential tools](#essential-tools)
    - [kubectl](#kubectl)
    - [krew](#krew)
    - [kubeconform](#kubeconform)
    - [kustomize](#kustomize)
    - [skaffold](#skaffold)
    - [telepresence](#telepresence)
  - [Deployment options](#deployment-options)
    - [Strategies](#strategies)
    - [Managed offering](#managed-offering)
      - [Traditional Kubernetes Offerings](#traditional-kubernetes-offerings)
      - [Serverless Kubernetes Offerings](#serverless-kubernetes-offerings)
    - [Enterprise offering](#enterprise-offering)
  - [Local environment - kubeadm](#local-environment---kubeadm)
    - [Installation - control plane node](#installation---control-plane-node)
    - [Installation - worker node](#installation---worker-node)
    - [Verification and customization](#verification-and-customization)
  - [Local Environment kind](#local-environment-kind)
    - [Installation](#installation)
    - [Cluster creation](#cluster-creation)
    - [Cluster verification](#cluster-verification)
    - [Cluster customization](#cluster-customization)
      - [Kubernetes version](#kubernetes-version)
      - [Node scaling](#node-scaling)
    - [Cluster cleanup](#cluster-cleanup)
    - [Kubeconfig](#kubeconfig)
  - [Local Environment minikube](#local-environment-minikube)
    - [Installation](#installation-1)
    - [Cluster creation](#cluster-creation-1)
    - [Cluster verification](#cluster-verification-1)
    - [Cluster customization](#cluster-customization-1)
      - [Kubernetes version](#kubernetes-version-1)
      - [Resouce allocation](#resouce-allocation)
      - [Node scaling](#node-scaling-1)
      - [Container Network Interface](#container-network-interface)
      - [Addons](#addons)
    - [Cluster cleanup](#cluster-cleanup-1)
    - [Kubeconfig](#kubeconfig-1)
  - [Quick start](#quick-start)
  - [Resources](#resources)
  - [Deployments](#deployments)
    - [Setup](#setup)
    - [Scaling](#scaling)
    - [Cleanup](#cleanup)
  - [Services](#services)
  - [Ingress](#ingress)
  - [Namespaces](#namespaces)
    - [Formatting](#formatting)
    - [Sorting](#sorting)
    - [Filtering](#filtering)
  - [Labels](#labels)
    - [Create](#create)
    - [Display](#display)
    - [Update](#update)
    - [Delete](#delete)
    - [Filter](#filter)
  - [Application Health Check](#application-health-check)
  - [Application Upgrades](#application-upgrades)
  - [Application Rollback](#application-rollback)
  - [Basic Troubleshooting](#basic-troubleshooting)
  - [Helm](#helm)
    - [Installation - Linux binary](#installation---linux-binary)
    - [Installation - rpm package](#installation---rpm-package)
    - [Public charts](#public-charts)
      - [Artifact Hub](#artifact-hub)
      - [Customization](#customization)
      - [Versions](#versions)
      - [Cleanup](#cleanup-1)
    - [Custom charts](#custom-charts)
      - [Creation](#creation)
      - [Customization - ConfigMap object](#customization---configmap-object)
      - [Customization - Secret object](#customization---secret-object)
      - [Rollbacks](#rollbacks)
      - [Templating](#templating)
  - [Tips](#tips)
    - [Hyper-V vSwitch Configuration](#hyper-v-vswitch-configuration)
    - [Minikube Docker Configuration](#minikube-docker-configuration)
    - [Minikube Tunnel Configuration](#minikube-tunnel-configuration)
    - [Play with K8s](#play-with-k8s)
    - [Kubectl Configuration](#kubectl-configuration)


## Documentation

- [Minikube](https://minikube.sigs.k8s.io/)
- [kind](https://kind.sigs.k8s.io/)
- [helm](https://helm.sh/)
- [Play With k8s](https://labs.play-with-k8s.com/)
- [CNCF Trailmap](https://github.com/cncf/trailmap)

## Essential tools

### kubectl

```bash
release=$(curl -L -s https://dl.k8s.io/release/stable.txt)
sudo curl -L -o /usr/local/bin/kubectl "https://dl.k8s.io/release/$release/bin/linux/amd64/kubectl"
sudo chmod +x /usr/local/bin/kubectl
kubectl version --version
```

### krew

[Krew](https://github.com/kubernetes-sigs/krew) is a plugin manager for kubectl.

Download and install krew.

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Update your shell configuration. Example below for bash.

```bash
# Create custom bash configuration
mkdir ~/.bashrc.d
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"'| tee ~/.bashrc.d/krew.conf

# Apply the change
source ~/.bashrc
```

Verify installation

```bash
kubectl krew update
Updated the local copy of plugin index.
```

To install a plugin use the `install` option.

```bash
kubectl krew install tree
```

Some other popular plugins include `sniff` which is useful for capturing network traffic.

### kubeconform

[Kubeconform](https://github.com/yannh/kubeconform) is a Kubernetes manifest validation tool. Incorporate it into your CI, or use it locally to validate your Kubernetes configuration!

```bash
# Retrieve the latest binary release number
version=$(curl --silent "https://api.github.com/repos/yannh/kubeconform/releases/latest" | jq -r .'tag_name')

# Download and install the latest binary release
curl -sL https://github.com/yannh/kubeconform/releases/download/$version/kubeconform-linux-amd64.tar.gz \
      | sudo tar -xz -C /usr/local/bin kubeconform

# Verify installation
kubeconform -v
v0.6.7
```

### kustomize

[kustomize](https://github.com/kubernetes-sigs/kustomize) lets you customize raw, template-free YAML files for multiple purposes, leaving the original YAML untouched and usable as is. Useful when migrating Docker `Dockerfile` to K8s `manifest`.

```bash
# Retrieve the latest binary release number
version=$(curl --silent "https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest" | jq -r .'tag_name' | cut -d"/" -f2)

# Download and install the latest binary release
curl -sL https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2F${version}/kustomize_${version}_linux_amd64.tar.gz \
      | sudo tar -xz -C /usr/local/bin kustomize

# Verify the installation
kustomize version
v5.4.3
```

### skaffold

[skaffold](https://github.com/GoogleContainerTools/skaffold) is a cli tool that handles the workflow for building, pushing and deploying your applications.

```bash
# Retrieve the latest binary release number
version=$(curl --silent "https://api.github.com/repos/GoogleContainerTools/skaffold/releases/latest" | jq -r .'tag_name')

# Download and install the latest binary release
sudo curl -sLo /usr/local/bin/skaffold https://github.com/GoogleContainerTools/skaffold/releases/download/${version}/skaffold-linux-amd64

sudo chmod +x /usr/loca/bin/skaffold

# Verify the installation
skaffold version
v2.13.2
```

### telepresence

[telepresence](https://github.com/telepresenceio/telepresence)

```bash

```

## Deployment options

### Strategies

| Kubernetes Offering | Automates Control Plane Provisioning | Automates Worker Node Provisioning and Joining | Automates Networking and Storage Configuration | Compliant with regulations | Privides additional Enterprise Grade Features |
|-|-|-|-|-|-|
| Cluster only | Y | | | | |
| Fully automated | Y | Y | Y | | |
| Kuberentes native | Y | Y | | | |
| Managed traditional | Y | Y | Y | | |
| Managed serverless | Y | Y | Y | | |
| Enterprise | Y | Y | Y | Y | Y |

### Managed offering

#### Traditional Kubernetes Offerings

- Manage the control plane
- You are responsible for joining nodes to your control plane usually
- You pay for worker nodes separately
- Examples include Azure Kubernetes Service, Elastic Kubernetes Service, Google Kubernetes Engine

#### Serverless Kubernetes Offerings

- Handle control plane, worker node registration, and autoscaling - you just bring your apps
- You control notrhing, providers handles everything
- Examples, AWS Fargate, GKE Autopilot, AKS ACI

### Enterprise offering

- Brings the benefit of managed offering into you private cloud
- Signifacntly more flexible
- Compliant with regulations
- Enterprise ready customizations e.g. networking and private registries
- Examples, Rancher Kubernetes Enginer, VMware Tanzu Kubernetes Grid, Red Hat OpenShift


## Local environment - kubeadm

The `./Vagrantfiles` folder contains a Vagrantfile that defines two virtual machines based on `Ubuntu 22.04`. The `README.md` in this folder contains further instructions on how to create cluster using `kubeadm`.

### Installation - control plane node

Start on control plane `cp` node by updating package list and upgrading all packages. Afterwards install the following packages.

```bash
# Update package list and upgrade all packages
sudo apt-get update && sudo apt-get upgrade -y

# Install common dependencies
sudo apt-get install -y curl \
                        apt-transport-https \
                        wget software-properties-common \
                        lsb-release \
                        ca-certificates
```

Update the system settings.

```bash
# Optionally disable swap
swapoff -a

# Load modules
modprobe overlay
modprobe br_netfilter

# Update kernel networking settings
cat << EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Ensure changes are applied
sudo sysctl --system
```

Download Docker key and configure its repository.

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install and configure containerd.io

```bash
sudo apt-get update && sudo apt-get install -y containerd.io
sudo cp /etc/containerd/config.toml{,.$(date +%Y%m%d)}
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
sudo systemctl restart containerd
```

Download Kubernetes key and configure its repository.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo \
"deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
```

Install kubernetes components for a specific version. To prevent automatic updates we will use `apt-mark`

```bash
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.1-1.1 kubelet=1.29.1-1.1 kubectl=1.29.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

Update local DNS entries

```bash
eth0_addr=$(ip -4 a show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

echo $eth0_addr k8s$(hostname) | sudo tee -a /etc/hosts
```

Create configuration file for the cluster.

```bash
cat << EOF | sudo tee /root/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.29.1
controlPlaneEndpoint: "k8scp:6443"
networking:
  podSubnet: 192.168.0.0/16
EOF
```

Initialize cp. At the end of the ouput, details for joining workers to cluster will be displayed.

```bash
sudo kubeadm init --config=/root/kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
[init] Using Kubernetes version: v1.29.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

...[ output ommited ]...

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8scp:6443 --token eb8gyw.fqmahgfypgdxnp3s \
        --discovery-token-ca-cert-hash sha256:5401ca9ef72cfd506ae9ba29e356dc462b4c0df8066f23f75d0896339c2e488b \
        --control-plane --certificate-key 4b997d5da1bcf4d4ad1a745251b4d8f8476373d0b981370a680584d0432a9d43

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8scp:6443 --token eb8gyw.fqmahgfypgdxnp3s \
        --discovery-token-ca-cert-hash sha256:5401ca9ef72cfd506ae9ba29e356dc462b4c0df8066f23f75d0896339c2e488b 

```

Copy the kube config to user's home/

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Setting up pod network

```bash
# Generate Cilium configuration
helm repo add cilium https://helm.cilium.io/
helm repo update
helm template cilium cilium/cilium --version 1.14.1 \
  --namespace kube-system > cilium-cni.yaml

# Apply the config
kubectl apply -f cilium-cni.yaml
```

Some nice to haves.

```bash
sudo apt-get install -y bash-completion
source <(kubectl completion bash)
cp $HOME/.bashrc{,.$(date +%Y%m%d)}
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
```

### Installation - worker node

Continue on worker `worker` node by updating package list and upgrading all packages. Afterwards install the following packages.

```bash
# Update package list and upgrade all packages
sudo apt-get update && sudo apt-get upgrade -y

# Install common dependencies
sudo apt-get install -y curl \
                        apt-transport-https \
                        wget software-properties-common \
                        lsb-release \
                        ca-certificates
```

Update the system settings.

```bash
# Optionally disable swap
swapoff -a

# Load modules
modprobe overlay
modprobe br_netfilter

# Update kernel networking settings
cat << EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Ensure changes are applied
sudo sysctl --system
```

Download Docker key and configure its repository.

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install and configure containerd.io

```bash
sudo apt-get update && sudo apt-get install -y containerd.io
sudo cp /etc/containerd/config.toml{,.$(date +%Y%m%d)}
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
sudo systemctl restart containerd
```

Download Kubernetes key and configure its repository.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo \
"deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
```

Install kubernetes components for a specific version. To prevent automatic updates we will use `apt-mark`

```bash
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.1-1.1 kubelet=1.29.1-1.1 kubectl=1.29.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

From `cp` node retrieve the token.

```bash
sudo kubeadm token create --print-join-command
```

Back on `worker node` update hosts file.

```bash
# Retrieve the cp primary ip address and update hosts on worker
echo 192.168.121.251 k8scp | sudo tee -a /etc/hosts
```

Join the cluster

```bash
sudo kubeadm join k8scp:6443 --token h4hzoq.4bi2yi8mkfw9695q --discovery-token-ca-cert-hash sha256:5401ca9ef72cfd506ae9ba29e356dc462b4c0df8066f23f75d0896339c2e488b | tee kubeadm-join.out
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### Verification and customization

Verify from `cp` node:

```bash
kubectl get nodes
NAME     STATUS   ROLES           AGE   VERSION
cp       Ready    control-plane   65m   v1.29.1
worker   Ready    <none>          52s   v1.29.1

kubectl describe node cp
```

Allow `cp` to run non-infra pods.

```bash
kubetl describe node | grep -i taint

kubectl taint nodes --all node-role.kubernetes.io/control-plane-

kubectl describe node | grep -i taint
Taints:             <none>
Taints:             <none>
```

Verify DNS and Cilium.

```bash
kubectl get pods --all-namespaces
kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   cilium-h2tzt                       1/1     Running   0             46m
kube-system   cilium-jv6l7                       1/1     Running   0             24m
kube-system   cilium-operator-56bdb99ff6-4vctm   1/1     Running   0             46m
kube-system   cilium-operator-56bdb99ff6-x4ltf   1/1     Running   0             46m
kube-system   coredns-76f75df574-qn7m6           1/1     Running   0             87m
kube-system   coredns-76f75df574-t2b87           1/1     Running   0             87m
kube-system   etcd-cp                            1/1     Running   0             88m
kube-system   kube-apiserver-cp                  1/1     Running   0             88m
kube-system   kube-controller-manager-cp         1/1     Running   0             88m
kube-system   kube-proxy-2sdnf                   1/1     Running   0             24m
kube-system   kube-proxy-xczv5                   1/1     Running   0             87m
kube-system   kube-scheduler-cp                  1/1     Running   1 (55m ago)   88m
```

 on cp

```bash
sudo crictl config --set \runtime-endpoint=unix:///run/containerd/containerd.sock \--set image-endpoint=unix:///run/containerd/containerd.sock
```

on worker

```bash
sudo crictl config --set \runtime-endpoint=unix:///run/containerd/containerd.sock \--set image-endpoint=unix:///run/containerd/containerd.sock
```

```bash
sudo cat /etc/crictl.yaml

runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false

```

Deploy simple app

```bash
kubectl create deployment nginx --image=nginx

kubectl get deployments

kubectl get deployment nginx -o yaml

kubectl describe deployment nginx

kubectl get events
```

Remove Deployment

```bash
kubectl delete deployment nginx
```

## Local Environment kind

Kind lets you create Kubernetes clusters locally inside Docker. It is great for testing on local developer machine.

### Installation

```bash
# Retrieve the latest binary release number
version=$(curl --silent "https://api.github.com/repos/kubernetes-sigs/kind/releases/latest" | jq -r .'tag_name')

# Download the latest binary release
curl -LO https://github.com/kubernetes-sigs/kind/releases/download/$version/kind-linux-amd64

# Install the latest binary release
sudo install kind-linux-amd64 /usr/local/bin/kind && rm kind-linux-amd64
```

To upgrade minikube to latest version, just repeat the above steps again.


### Cluster creation

Creating a local cluster with kind is as simple as running the following command.

```bash
kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.31.0)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind
```

This will create a cluster called `kind` with latest kubernetes version supported by kind.

### Cluster verification

```bash
# Verify node status
kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION           CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   4m52s   v1.31.0   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.10.7-200.fc40.x86_64   containerd://1.7.18

# Verify all pods
kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-6f6b679f8f-fcxk5                     1/1     Running   0          5m16s
kube-system          coredns-6f6b679f8f-l98tq                     1/1     Running   0          5m16s
kube-system          etcd-kind-control-plane                      1/1     Running   0          5m22s
kube-system          kindnet-zr8ml                                1/1     Running   0          5m16s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          5m22s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          5m21s
kube-system          kube-proxy-mjnss                             1/1     Running   0          5m16s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          5m21s
local-path-storage   local-path-provisioner-57c5987fd4-85g4h      1/1     Running   0          5m16s

# Verify pod deployment
kubectl run --rm --stdin --image=hello-world \
            --restart=Never --request-timeout=30 \
            hw-pod

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
[ ... Output omitted ...]
```

### Cluster customization

Kind provides number of options to customize the cluster deployment. You can define these custom values using cli options or inside a configuration file.

#### Kubernetes version

For instance, using cli options you can define a name and desired kubernetes version for the cluster.

```bash
kind create cluster --name kind --image=kindest/node:v1.21.12
```

#### Node scaling

Create a cluster configuration file.

```bash
cat <<EOF > cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Next, start a cluster using the `--config` option.

```bash
# Create multi node cluster
kind create cluster --name multinode --config ./cluster.yaml

# Verify node status
kubectl get nodes
NAME                      STATUS   ROLES           AGE   VERSION
multinode-control-plane   Ready    control-plane   41s   v1.31.0
multinode-worker          Ready    <none>          28s   v1.31.0
multinode-worker2         Ready    <none>          28s   v1.31.0
```

### Cluster cleanup

To delete the kind cluster use the following command.

```bash
kind delete cluster --name multinode
Deleting cluster "multinode" ...
Deleted nodes: ["multinode-worker2" "multinode-control-plane" "multinode-worker"]
```

### Kubeconfig

By default when you start a new cluster, kind ensures that your default kubeconfig `~/.kube/config` is updated. In case you prefer to keep the configuration files in separate files for each environment, you can do so by updating the `KUBECONFIG` variable first.

```bash
# Backup your current config file, just in case
cp ~/.kube/config{,.$(date +%Y%m%d)}

# Sinkhole it to /dev/null
ln -sf /dev/null ~/.kube/config

# Update the KUBECONFIG env var
export KUBECONFIG=~/.kube/kind

# Regenerated the configuration for minikube
kind export kubeconfig

# Test access to cluster
kubectl cluster-info > /dev/null && echo "Exit code 0 so we are OK"
```

## Local Environment minikube

### Installation

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

To upgrade minikube to latest version, just repeat the above steps again.

### Cluster creation

```bash
minikube start
```

### Cluster verification

```bash
# Verify node status
kubectl get nodes -o wide
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION           CONTAINER-RUNTIME
demo   Ready    control-plane   12s   v1.30.0   192.168.49.2   <none>        Ubuntu 22.04.4 LTS   6.10.7-200.fc40.x86_64   docker://26.1.1

# Verify all pods
kubectl get pods -A
NAMESPACE     NAME                           READY   STATUS    RESTARTS        AGE
kube-system   coredns-7db6d8ff4d-p7645       1/1     Running   0               3m8s
kube-system   etcd-demo                      1/1     Running   0               3m22s
kube-system   kube-apiserver-demo            1/1     Running   0               3m23s
kube-system   kube-controller-manager-demo   1/1     Running   0               3m22s
kube-system   kube-proxy-bn6lf               1/1     Running   0               3m9s
kube-system   kube-scheduler-demo            1/1     Running   0               3m22s
kube-system   storage-provisioner            1/1     Running   1 (2m38s ago)   3m21s

# Verify pod deployment
kubectl run --rm --stdin --image=hello-world \
            --restart=Never --request-timeout=30 \
            hw-pod
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
[ ... Output omitted ...]
```

### Cluster customization

#### Kubernetes version

For instance, using cli options you can define a name and desired kubernetes version for the cluster.

```bash
minikube start --kubernetes-version=v1.31.0

# Get node information
kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION           CONTAINER-RUNTIME
minikube   Ready    control-plane   67s   v1.31.0   192.168.49.2   <none>        Ubuntu 22.04.4 LTS   6.10.7-200.fc40.x86_64   docker://26.1.1
```

#### Resouce allocation

```bash
minikube start ...
```

#### Node scaling

```bash
# Add another node
minikube node add
```

#### Container Network Interface

[Kindnet](https://github.com/aojea/kindnet) is the default CNI plugin in for minikube. It is a simple and capable of assigning IP addresses to pods running in the cluster and it does not support network policies.

[Calico]() is another popular CNI which offers more capability than kindnet. The easiest way to install Calico for Minikube is to use minikube options when create cluster. Other methods include using a Manifest and Operator.

```bash
# Remove existing cluster
minikube delete -p demo

# Install calico
minikube start --network-plugin=cni --cni=calico

# Verify Calico
kubectl get pods -A -l 'k8s-app in (calico-kube-controllers,calico-node)'
NAMESPACE     NAME                                      READY   STATUS    RESTARTS        AGE
kube-system   calico-kube-controllers-ddf655445-whzbx   1/1     Running   1 (3m36s ago)   3m55s
kube-system   calico-node-t4jr2                         1/1     Running   0               3m55s
```

#### Addons

There are other useful addons such as `dashboard`, `metrics-server`, `ingress` and `ingress-dns`. Below is a brief description of these.

- dashboard: A web-based UI for managing and monitoring Kubernetes clusters.
- metrics-server: Collects resource usage (CPU/memory) data for autoscaling and monitoring.
- ingress: Manages external HTTP/HTTPS access to services via an NGINX Ingress Controller.
- ingress-DNS: Provides automatic DNS resolution for services exposed through the Ingress addon.

You can enable the specific addon by using the `--addons` option.

```bash
minikube start \
         --addons=dashboard \
         --addons=metrics-server \
         --addons=ingress \
         --addons=ingress-dns
```

### Cluster cleanup

To delete the minikube cluster use the following command.

```bash
minikube delete
```

### Kubeconfig

By default when you start a new cluster, minikube ensures that your default kubeconfig `~/.kube/config` is updated. In case you prefer to keep the configuration files in separate files for each environment, you can do so by updating the `KUBECONFIG` variable first.

```bash
# Backup your current config file, just in case
cp ~/.kube/config{,.$(date +%Y%m%d)}

# Sinkhole it to /dev/null
ln -sf /dev/null ~/.kube/config

# Update the KUBECONFIG env var
export KUBECONFIG=~/.kube/minikube

# Regenerated the configuration for minikube
minikube update-context

# Test access to cluster
kubectl cluster-info > /dev/null && echo "Exit code 0 so we are OK"
```



## Quick start

Start by creating a new python virtual environment:

```bash
pyenv virtualenv 3.9.6 k8s-cookbook
```

Next, install required pip packages:

```bash
pip install --upgrade pip setuptools
pip install -r requirements-dev.txt
```

## Resources

To list all resources tha kubernetes provides we can use the `api-resources` option.

```bash
kubectl api-resources
```

## Deployments

To create a blank manifest file for a deployment you can use the `kubectl` with the `create` command.

```bash
# Generate the deployment template
kubectl create deployment explorecalifornia.com \
                          --dry-run=client \
                          --image localhost:5000/explorecalifornia.com \
                          --output=yaml \
                          | tee deployment.yaml
# Sanitize the deployment template
sed -i '/creationTimestamp/d' deployment.yaml
```

Use `port-forward` option to quickly test the new deployment.

```bash
kubectl port-forward deployment/explorecalifornia.com 8080:80
```

### Setup

Create new deployment from hello-go image:

```bash
kubectl create deployment hello-go
```

Check status of deployment:

```bash
kubectl get deployment hello-go
```

Check status of pods:

```bash
kubectl get pods
```

Get more details about pod:

```bash
kubectl describe pod hello-go
```

Modify deployment:

```bash
kubectl edit deployment hello-go
```

Expose deployment:

```bash
kubectl expose deployment hello-go --type=LoadBalancer --port=8180
```

Check the service:

```bash
kubectl get service hello-go
```

Interact with service (when using minikube):

```bash
minikube service hello-go
```

Display instances logs:

```bash
kubectl logs -l app=hello-go
```

Get all resources:

```bash
kubectl get all
```


### Scaling

Increase number of instances to 4

```bash
kubectl scale deployments/hello-go --replicas=4
```

Check deployment:

```bash
kubectl get deployment hello-go
```

Get logs by pod using for loop:

```bash
for pod in $(kubectl get po -l app=hello-go -oname); do echo $pod; kubectl logs $pod; done;
```

Get logs by pod using labels:

```bash
kubectl logs -l app=hello-go --all-containers --prefix
```


### Cleanup

Delete the service:

```bash
kubectl delete service hello-go
```

Delete the Deployment and associated pods:

```bash
kubectl delete deployment hello-go
```

Delete the container image:

```bash
docker rmi hello-go
```

The default delete argument will wait for pods to gracefully shutdown. You also have an option to force this action:

```bash
kubectl delete deployment --grace-period=0 --force
```

## Services

To create a blank manifest file for a service you can use the `kubectl` with the 

```bash
# Generate the serivce template
kubectl create service clusterip \
                          explorecalifornia.com \
                          --dry-run=client \
                          --tcp=80:80 \
                          --output=yaml \
                          | tee service.yaml

# Sanitize the deployment template
sed -i '/creationTimestamp/d' service.yaml
yq eval '.metadata.name = "explorecalifornia-svc"' -i service.yaml
```

Apply the service manifest.

```bash
kubectl apply -f service.yaml
```

Verify all object with label `explorecalifornia.com`

```bash
kubectl get all -l app=explorecalifornia.com
```

Use the `port-forward` to quickly test the new service.

```bash
kubectl port-forward service/explorecalifornia-svc 8080:80
```

## Ingress

```bash
# Generate the ingress template
kubectl create ingress explorecalifornia.com \
                       --dry-run=client \
                       --rule="explorecalifornia.com/=explorecalifornia-svc:80"  \
                       --output=yaml \
                       | tee ingress.yaml

# Update the pathType from Exact to Prefix
yq eval '.spec.rules[0].http.paths[0].pathType = "Prefix"' -i ingress.yaml

# Remove the .status object
yq eval 'del(.status)' -i ingress.yaml

# Sanitize the ingress template
sed -i '/creationTimestamp/d' ingress.yaml
```

Verify the kind cluster config

```bash
yq '.nodes[].kubeadmConfigPatches[]' kind_config.yaml 
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    node-labels: "ingress-ready=true"
```

Deploy and verify NGINX ingress controller.

```bash
# Deploy 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Verify
kubectl get all -n ingress-nginx
```

Update `/etc/hosts` with record for `explorecalifornia.com`

```bash
127.0.0.1 explorecalifornia.com
```

Apply the ingress manifest.

```bash
kubectl apply -f ingress.yaml
```

Test the application

```bash
curl explorecalifornia.com
```

## Namespaces

Namespaces are used to organize resources in cluster. By default new applications are deployed in the `default` namespace. There are also some tthat kubernetes uses for its operation.

Create a sample infrastructure deployment:

```bash
kubectl apply -f sample-infrastrcuture.yaml
```

Display existing namespaces:

```bash
kubectl get namespaces
```

You should see output similar to this:

```bash
NAME              STATUS   AGE
auth              Active   112s
cart              Active   112s
catalog           Active   111s
default           Active   4d16h
infra             Active   111s
kube-node-lease   Active   4d16h
kube-public       Active   4d16h
kube-system       Active   4d16h
purchasing        Active   111s
quote             Active   111s
social            Active   111s
web               Active   112s
```

Display all pods from `cart` namespace:

```
kubectl get pods -n cart
```


### Formatting

You can change the default output format using the `-o` flag.

Using the wide format to display all pods in `cart` namespace:

```bash
kubectl get pods -o wide -n cart
```

You should see output similar to this:

```bash
NAME           READY   STATUS    RESTARTS      AGE   IP            NODE       NOMINATED NODE   READINESS GATES
cart-dev       1/1     Running   1 (80s ago)   11m   172.17.0.12   minikube   <none>           <none>
cart-prod      1/1     Running   1 (95s ago)   11m   172.17.0.6    minikube   <none>           <none>
cart-staging   1/1     Running   1 (73s ago)   11m   172.17.0.11   minikube   <none>           <none>
```

Display all pods in `cart` namespace using `json` output format:

```bash
kubectl get pods -o json -n cart
```

This output can be further parsed using `jq` tool:

```json

    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "v1",
            "kind": "Pod",
            "metadata": {
                "annotations": {
                    "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"application_type\":\"api\",\"dev-lead\":\"carisa\",\"env\":\"development\",\"release-version\":\"1.0\",\"team\":\"ecommerce\"},\"name\":\"cart-dev\",\"namespace\":\"cart\"},\"spec\":{\"containers\":[{\"image\":\"karthequian/ruby:latest\",\"name\":\"cart\"}]}}\n"
                },
                "creationTimestamp": "2022-08-07T11:28:17Z",
                "labels": {
                    "application_type": "api",
                    "dev-lead": "carisa",
                    "env": "development",
                    "release-version": "1.0",
                    "team": "ecommerce"
...                },
[ Omitted for brevity ]
...
```

Display all pods in `cart` namespace using `yaml` output format:

```bash
kubectl get pods -o yaml -n cart
```

The output is now formwatted using yaml:

```yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"application_type":"api","dev-lead":"carisa","env":"development","release-version":"1.0","team":"ecommerce"},"name":"cart-dev","namespace":"cart"},"spec":{"containers":[{"image":"karthequian/ruby:latest","name":"cart"}]}}
    creationTimestamp: "2022-08-07T11:28:17Z"
    labels:
      application_type: api
      dev-lead: carisa
      env: development
      release-version: "1.0"
      team: ecommerce
...
[ Omitted for brevity ]
...
```


### Sorting

In order to sort based on various properties, use the `--sort-by` argument.

```bash
kubectl get pods --sort-by=.metadata.name --all-namespaces | head -n 6
```

The list of pods is sorted by pod name instead of namespace:

```bash
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
cart          cart-dev                           1/1     Running   1 (6m30s ago)   17m
cart          cart-prod                          1/1     Running   1 (6m45s ago)   17m
cart          cart-staging                       1/1     Running   1 (6m23s ago)   17m
catalog       catalog-dev                        1/1     Running   1 (6m ago)      17m
catalog       catalog-prod                       1/1     Running   1 (6m4s ago)    17m
```


### Filtering

In order to filter a property, you can use output type of `jsonpath`:

```bash
kubectl get pods -o=jsonpath="{..image}" -l env=staging -n cart
```

The output from this command shows that we are using the following images in `cart` namespace in `staging` environment.

```bash
karthequian/ruby:latest karthequian/ruby:latest
```


## Labels

Labels are used to organize resources in the K8s cluster. You can define labels in the resource definition during deployment time.

### Create

```yml
kind: Pod
metadata:
  name: helloworld
  labels:
    env: production
    author: maros
    application_type: ui
    release-version: "1.0"
```


### Display

After the resource has been create you can display label values with `--show-labels` argument.

Display pods with associated labels:

```bash
kubectl get pods --show-labels
```

The `LABELS` column is displayed along all the applied labels:

```bash
NAME         READY   STATUS    RESTARTS   AGE   LABELS
helloworld   1/1     Running   0          84s   application_type=ui,author=maros,env=production,release-version=1.0
```


### Update

You can also create or update labels during runtime with `label` resource.

Create or update label value:

```bash
kubectl label pod/helloworld app_id=1092134
kubectl label pod/helloworld app=helloworldapp
```


### Delete

Delete a label:

```bash
kubectl label pod/helloworld app-
```


### Filter

In order to filter based on labels, create a sample infrastructure.

Create a bunch of sample pods:

```bash
kubectl apply -f https://raw.githubusercontent.com/karthequian/Kubernetes/master/04_02_Searching_for_labels/sample-infrastructure-with-labels.yml
```

Display pods with associated labels:

```bash
NAME               READY   STATUS    RESTARTS   AGE    LABELS
cart-dev           1/1     Running   0          116s   application_type=api,dev-lead=carisa,env=development,release-version=1.0,team=ecommerce
cart-prod          1/1     Running   0          116s   application_type=api,dev-lead=carisa,env=production,release-version=1.0,team=ecommerce
cart-staging       1/1     Running   0          116s   application_type=api,dev-lead=carisa,env=staging,release-version=1.0,team=ecommerce
catalog-dev        1/1     Running   0          116s   application_type=api,dev-lead=daniel,env=development,release-version=4.0,team=ecommerce
catalog-prod       1/1     Running   0          116s   application_type=api,dev-lead=daniel,env=production,release-version=4.0,team=ecommerce
catalog-staging    1/1     Running   0          116s   application_type=api,dev-lead=daniel,env=staging,release-version=4.0,team=ecommerce
homepage-dev       1/1     Running   0          117s   application_type=ui,dev-lead=karthik,env=development,release-version=12.0,team=web
homepage-prod      1/1     Running   0          116s   application_type=ui,dev-lead=karthik,env=production,release-version=12.0,team=web
homepage-staging   1/1     Running   0          116s   application_type=ui,dev-lead=karthik,env=staging,release-version=12.0,team=web
login-dev          1/1     Running   0          116s   application_type=api,dev-lead=jim,env=development,release-version=1.0,team=auth
login-prod         1/1     Running   0          116s   application_type=api,dev-lead=jim,env=production,release-version=1.0,team=auth
login-staging      1/1     Running   0          116s   application_type=api,dev-lead=jim,env=staging,release-version=1.0,team=auth
ordering-dev       1/1     Running   0          116s   application_type=backend,dev-lead=chen,env=development,release-version=2.0,team=purchasing
ordering-prod      1/1     Running   0          116s   application_type=backend,dev-lead=chen,env=production,release-version=2.0,team=purchasing
ordering-staging   1/1     Running   0          116s   application_type=backend,dev-lead=chen,env=staging,release-version=2.0,team=purchasing
quote-dev          1/1     Running   0          116s   application_type=api,dev-lead=amy,env=development,release-version=2.0,team=ecommerce
quote-prod         1/1     Running   0          116s   application_type=api,dev-lead=amy,env=production,release-version=1.0,team=ecommerce
quote-staging      1/1     Running   0          116s   application_type=api,dev-lead=amy,env=staging,release-version=2.0,team=ecommerce
social-dev         1/1     Running   0          116s   application_type=api,dev-lead=carisa,env=development,release-version=2.0,team=marketing
social-prod        1/1     Running   0          116s   application_type=api,dev-lead=marketing,env=production,release-version=1.0,team=marketing
social-staging     1/1     Running   0          116s   application_type=api,dev-lead=marketing,env=staging,release-version=1.0,team=marketing
```

Search with `selector` argument.

Display pods in production environment:

```bash
kubectl get pods --selector env=production
```

```bash
NAME            READY   STATUS    RESTARTS   AGE
cart-prod       1/1     Running   0          4m6s
catalog-prod    1/1     Running   0          4m6s
homepage-prod   1/1     Running   0          4m6s
login-prod      1/1     Running   0          4m6s
ordering-prod   1/1     Running   0          4m6s
quote-prod      1/1     Running   0          4m6s
social-prod     1/1     Running   0          4m6s
```

Display pods with multiple labels:

```bash
kubectl get pods --selector dev-lead=carisa,env=development
```

```bash
NAME         READY   STATUS    RESTARTS   AGE
cart-dev     1/1     Running   0          6m46s
social-dev   1/1     Running   0          6m46s
```

Display pods with multiple labels - negation:

```bash
kubectl get pods --selector dev-lead!=carisa,env=development --show-labels
```

```bash
NAME           READY   STATUS    RESTARTS   AGE     LABELS
catalog-dev    1/1     Running   0          9m52s   application_type=api,dev-lead=daniel,env=development,release-version=4.0,team=ecommerce
homepage-dev   1/1     Running   0          9m53s   application_type=ui,dev-lead=karthik,env=development,release-version=12.0,team=web
login-dev      1/1     Running   0          9m52s   application_type=api,dev-lead=jim,env=development,release-version=1.0,team=auth
ordering-dev   1/1     Running   0          9m52s   application_type=backend,dev-lead=chen,env=development,release-version=2.0,team=purchasing
quote-dev      1/1     Running   0          9m52s   application_type=api,dev-lead=amy,env=development,release-version=2.0,team=ecommerce
```

Instead of `--selector` argument you can also use the `-l` argument.

Display pods with certain release-version value (in or notin)

```bash
kubectl get pods -l 'release-version in (1.0,2.0)'
```

```bash
NAME               READY   STATUS    RESTARTS      AGE
cart-dev           1/1     Running   1 (33s ago)   12m
cart-prod          1/1     Running   1 (63s ago)   12m
cart-staging       1/1     Running   1 (67s ago)   12m
login-dev          1/1     Running   1 (88s ago)   12m
login-prod         1/1     Running   1 (71s ago)   12m
login-staging      1/1     Running   1 (94s ago)   12m
ordering-dev       1/1     Running   1 (24s ago)   12m
ordering-prod      1/1     Running   1 (16s ago)   12m
ordering-staging   1/1     Running   1 (20s ago)   12m
quote-dev          1/1     Running   1 (57s ago)   12m
quote-prod         1/1     Running   1 (30s ago)   12m
quote-staging      1/1     Running   1 (79s ago)   12m
social-dev         1/1     Running   1 (75s ago)   12m
social-prod        1/1     Running   1 (54s ago)   12m
social-staging     1/1     Running   1 (38s ago)   12m
```

You can also use labels to delete a resource.

Delete all pods owned by carisa in dev environment:

```bash
kubectl delete pod -l dev-lead=carisa,env=development
```

```bash
pod "cart-dev" deleted
pod "social-dev" deleted
```


## Application Health Check

The `readinessProbe` and `livenessProbe` defined in deployment specification are used for application level healthchecks.

Apply a new deployment:

```bash
kubectl apply -f helloworld-with-probes.yaml
```

Display the deployment status:

```bash
kubectl get deployment/helloworld-deployment-with-probes
```

```bash
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment-with-probes   1/1     1            1           27m
```

Display the replicaset status:

```bash
kubectl get replicaset
```

```bash
NAME                                           DESIRED   CURRENT   READY   AGE
helloworld-deployment-with-probes-644c46c778   1         1         1       30m
```


## Application Upgrades

Apply the *black* version of the deployment:

```bash
kubectl apply -f helloworld-black.yaml
```

Retrieve the Node Port:

```bash
NODE_PORT=$(kubectl get service navbar-service -o jsonpath="{.spec.ports[].nodePort}")
```

Retrieve the Node Address:

```bash
NODE_ADDRESS=$(kubectl get nodes -o=jsonpath="{.items[].status.addresses[].address}")
```

Verify the application:

```bash
curl --head http://$NODE_ADDRESS:$NODE_PORT
```

In case of success you should see the following response:

```bash
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Wed, 17 Aug 2022 10:10:09 GMT
Content-Type: text/html
Content-Length: 4216
Last-Modified: Sun, 15 Oct 2017 23:49:41 GMT
Connection: keep-alive
ETag: "59e3f415-1078"
Accept-Ranges: bytes
```

When you view this application in browser, you will see that the navbar background color is black. Now update this deployment with new image version:

```bash
kubectl set image deployment/navbar-deployment helloworld=karthequian/helloworld:blue
```

This will create a new replicaset `navbar-deployment-85ffd45b97` which has a new image defined.

```bash
kubectl get rs
```

```bash
NAME                           DESIRED   CURRENT   READY   AGE
navbar-deployment-65cf9bd74d   0         0         0       53m
navbar-deployment-85ffd45b97   3         3         3       22m
```

## Application Rollback

```bash
kubectl rollout undo deployment navbar-deployment
```


## Basic Troubleshooting

```bash
kubectl describe <pod-name>
```

```bash
kubectl logs <pod-name>
```

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

## Helm

Helm is the de-facto package management solution for kubernetes.

### Installation - Linux binary

```bash
# Retrieve the latest binary release number
version=$(curl --silent "https://api.github.com/repos/helm/helm/releases/latest" | jq -r .'tag_name')

# Download and install the latest binary release
curl -L https://get.helm.sh/helm-$version-linux-amd64.tar.gz \
     | sudo tar -xz --strip-components=1 -C /usr/local/bin linux-amd64/helm

# Verify the binary
helm version
version.BuildInfo{Version:"3.15.4", GitCommit:"fa9efb07d9d8debbb4306d72af76a383895aa8c4", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

### Installation - rpm package

```bash
# Update package list
sudo dnf update

# Install rpm package
sudo dnf install -t helm

# Verify the installation
helm version
version.BuildInfo{Version:"3.15.4", GitCommit:"fa9efb07d9d8debbb4306d72af76a383895aa8c4", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

### Public charts

#### Artifact Hub

Artifact Hub provides ready to use helm charts to deploy popular application easily. One of the popular charts include the `kube-state-metrics` available from bitnami repository. In order to deploy this chart we need to initialize the repository first.

```bash
# Initialize Bitnami Helm Chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update helm repositories to retrieve latest version of charts
helm repo update

# List available repository
helm repo list
bitnami https://charts.bitnami.com/bitnami
```

Next, we install `kube-state-metrics` chart in a new namespace.

```bash
# Create a new namespace
kubectl create namespace metrics

# Install the chart in metrics namespace
helm install kube-state-metrics bitnami/kube-state-metrics -n metrics

# List all objects that are part of metrics namespaces
kubectl get all -n metrics
NAME                                      READY   STATUS    RESTARTS   AGE
pod/kube-state-metrics-6b5b49c49d-wntgq   1/1     Running   0          8h

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kube-state-metrics   ClusterIP   10.109.60.234   <none>        8080/TCP   8h

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kube-state-metrics   1/1     1            1           8h

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/kube-state-metrics-6b5b49c49d   1         1         1       8h

kubectl logs pod/kube-state-metrics-6b5b49c49d-wntgq -n metrics
I0910 11:36:22.413959       1 wrapper.go:120] "Starting kube-state-metrics"
W0910 11:36:22.414587       1 client_config.go:659] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0910 11:36:22.415001       1 server.go:218] "Used resources" resources=["statefulsets","volumeattachments","configmaps","mutatingwebhookconfigurations","horizontalpodautoscalers","services","pods","namespaces","leases","networkpolicies","deployments","certificatesigningrequests","ingresses","poddisruptionbudgets","resourcequotas","storageclasses","daemonsets","endpoints","replicasets","replicationcontrollers","nodes","jobs","limitranges","persistentvolumeclaims","persistentvolumes","cronjobs","secrets"]
I0910 11:36:22.415076       1 types.go:227] "Using all namespaces"
[ ... output omitted...]
```

Finally we use `port-forward` feature to access the `kube-state-metrics` service.

```bash
# Use port-forward option to expose kube-state-metrics service
kubectl port-forward svc/kube-state-metrics -n metrics 8080:8080 &

# Retrieve data from /metrics path
curl -L localhost:8080/metrics | head
# Retrieve the content of metrics endpoint
# HELP kube_configmap_annotations Kubernetes annotations converted to Prometheus labels.
# TYPE kube_configmap_annotations gauge
# HELP kube_configmap_labels [STABLE] Kubernetes labels converted to Prometheus labels.
# TYPE kube_configmap_labels gauge
# HELP kube_configmap_info [STABLE] Information about configmap.
# TYPE kube_configmap_info gauge
kube_configmap_info{namespace="kube-system",configmap="coredns"} 1
kube_configmap_info{namespace="kube-system",configmap="extension-apiserver-authentication"} 1
kube_configmap_info{namespace="kube-system",configmap="kube-apiserver-legacy-service-account-token-tracking"} 1
kube_configmap_info{namespace="kube-system",configmap="kube-proxy"} 1
[ ... output omitted ...]

# Exit the port-forward
kill %1
[1]+  Terminated              kubectl port-forward service/kube-state-metrics -n metrics 8080:8080
```

#### Customization

Retrieve information for a given chart.

```bash
# Display chart metadata
helm show chart bitnami/kube-state-metrics


# Display default values
helm show values bitnami/kube-state-metrics | yq .
```

#### Versions

```bash
# Retrieve detauls about helm deployment in metrics namespace
helm ls -n metrics
NAME              	NAMESPACE	REVISION	UPDATED                                 	STATUS  	CHART                    	APP VERSION
kube-state-metrics	metrics  	1       	2024-09-10 13:35:58.267339455 +0200 CEST	deployed	kube-state-metrics-4.2.13	2.13.0 

# Set specific version
helm upgrade kube-state-metrics bitnami/kube-state-metrics --version 4.2.12 -n metrics
Release "kube-state-metrics" has been upgraded. Happy Helming!
NAME: kube-state-metrics
LAST DEPLOYED: Tue Sep 10 22:45:00 2024
NAMESPACE: metrics
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: kube-state-metrics
CHART VERSION: 4.2.12
APP VERSION: 2.13.0

** Please be patient while the chart is being deployed **

Watch the kube-state-metrics Deployment status using the command:

    kubectl get deploy -w --namespace metrics kube-state-metrics

kube-state-metrics can be accessed via port "8080" on the following DNS name from within your cluster:

    kube-state-metrics.metrics.svc.cluster.local

To access kube-state-metrics from outside the cluster execute the following commands:

    echo "URL: http://127.0.0.1:9100/"
    kubectl port-forward --namespace metrics svc/kube-state-metrics 9100:8080

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

Other popular charts include [ingress nginx](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx) and [datadog](https://artifacthub.io/packages/helm/datadog/datadog).

#### Cleanup

```bash
# Delete the chart
helm delete kube-state-metrics -n metrics
release "kube-state-metrics" uninstalled

# Delete the namespace
kubectl delete ns metrics
namespace "metrics" deleted
```

### Custom charts

#### Creation

Generate a new template Helm chart.

```bash
helm create sample-chart
cd $_
# Helm created the following file structure
tree
.
├── charts                        # Directory containing sub-charts or dependencies
├── Chart.yaml                    # Metadata file defining the chart's name, version, and dependencies
├── templates                     # Directory containing Kubernetes resource templates
│   ├── deployment.yaml           # Template for creating a Kubernetes Deployment resource
│   ├── _helpers.tpl              # Helper template file with reusable functions for other templates
│   ├── hpa.yaml                  # Template for creating a Horizontal Pod Autoscaler (HPA) resource
│   ├── ingress.yaml              # Template for creating a Kubernetes Ingress resource for traffic routing
│   ├── NOTES.txt                 # Plain text file with instructions or information displayed after installation
│   ├── serviceaccount.yaml       # Template for creating a Kubernetes ServiceAccount resource
│   ├── service.yaml              # Template for creating a Kubernetes Service resource
│   └── tests                     # Directory for Helm tests
│       └── test-connection.yaml  # Template for creating test resources, typically for validating the release
└── values.yaml                   # Default configuration values for the chart
```

#### Customization - ConfigMap object

Lets edit this boilerplate by removing all files from templates directory and creaing a new one called configmap.yaml wit the following content.

```bash
rm -rf templates/*
cat <<EOF > templates/configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap
data:
  port: "8080"
EOF
```

Install the current version of chart.

```bash
helm install first-chart .
NAME: first-chart
LAST DEPLOYED: Wed Sep 11 12:32:47 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Verify the newly created object.

```bash
kubectl get configmap first-chart-configmap
NAME                    DATA   AGE
first-chart-configmap   1      57s

kubectl describe configmap first-chart-configmap
Name:         first-chart-configmap
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: first-chart
              meta.helm.sh/release-namespace: default

Data
====
port:
----
8080
Events:  <none>
```

Lets update the `configmap.yaml` using `yq` in place editing.

```bash
yq eval '.data.allowTesting = "true" ' -i templates/configmap.yaml
```

The update file will have the following content:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap
data:
  port: "8080"
  allowTesting: "true"
```

To show how helm fill render the template use the `template` command.

```bash
helm template first-chart .
---
# Source: sample-chart/templates/configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap
data:
  port: "8080"
  allowTesting: "true"
```

Finally update the helm deployment.

```bash
# Upgrade the helm chart
helm upgrade first-chart .
Release "first-chart" has been upgraded. Happy Helming!
NAME: first-chart
LAST DEPLOYED: Wed Sep 11 12:40:11 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None

# Verify configmap value
kubectl describe configmap first-chart-configmap
Name:         first-chart-configmap
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: first-chart
              meta.helm.sh/release-namespace: default

Data
====
allowTesting:
----
true
port:
----
8080
Events:  <none>
```

#### Customization - Secret object

Create a new manifest for secret object. Remember the values need to be base64 encoded. By using cat with redirection it is possible to do this using echo with pipe to base64.

```bash
cat <<EOF > templates/secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: first-chart-secret
data:
  username: $(echo -n 'admin' | base64 )
  password: $(echo -n 'password' | base64 )
EOF
```
The resulting template file is:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: first-chart-secret
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

Render the template.

```bash
helm template first-chart .
---
# Source: sample-chart/templates/configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap
data:
  port: "8080"
  allowTesting: "true"
---
# Source: sample-chart/templates/secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: first-chart-secret
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

And deploy and verify.

```bash
# Deploy new revision
helm upgrade first-chart .

# Verify secret object
for key in username password; do
 echo "$key: $(kubectl get secret first-chart-secret -o jsonpath="{.data.$key}" | base64 --decode)"
done
# Response
username: admin
password: password
```

#### Rollbacks

Start by identifying all charts that have been deployed.

```bash
helm list
NAME            NAMESPACE       REVISION        UPDATED                                         STATUS          CHART                   APP VERSION
first-chart     default         4               2024-09-11 12:56:11.813988214 +0200 CEST        deployed        sample-chart-0.1.0      1.16.0  
```

Focus on `first-chart` deployment and look at its history.

```bash
helm history first-chart
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION                                                                                                                                                                                                 
1               Wed Sep 11 12:32:47 2024        superseded      sample-chart-0.1.0      1.16.0          Install complete                                                                                                                                                                                            
2               Wed Sep 11 12:40:11 2024        superseded      sample-chart-0.1.0      1.16.0          Upgrade complete                                                                                                                                                                                            
3               Wed Sep 11 12:54:54 2024        failed          sample-chart-0.1.0      1.16.0          Upgrade "first-chart" failed: failed to create resource: secret in version "v1" cannot be handled as a Secret: no kind "secret" is registered for version "v1" in scheme "pkg/api/legacyscheme/scheme.go:30"
4               Wed Sep 11 12:56:11 2024        deployed        sample-chart-0.1.0      1.16.0          Upgrade complete   
```

Rollback to previos version

```bash
helm rollback first-chart 2
Rollback was a success! Happy Helming!
```


#### Templating

Update the templates to use a value from `Chart.yaml`.

```bash
yq eval '.metadata.name = "first-chart-configmap-{{.Chart.Version}}" ' -i templates/configmap.yaml
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap-{{.Chart.Version}}
data:
  port: "8080"
  allowTesting: "true"
```

Render the value to verify and then deploy and verify again.

```bash
helm template first-chart . --show-only templates/configmap.yaml | yq .metadata.name
first-chart-configmap-0.1.0

helm upgrade first-chart .

kubectl get cm
NAME                          DATA   AGE
first-chart-configmap-0.1.0   2      19s
kube-root-ca.crt              1      44m
```

Update the templates to use an updated value from `values.yaml`.

```bash
# Display default replica count
yq '.replicaCount' ./values.yaml
1

# Update the count
yq eval '.replicaCount = 3' -i ./values.yaml
```

To reference this value in a template you can use the `{{ .Values.replicaCount }}`

Helm also support conditional statements.

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap-{{.Chart.Version}}
data:
  port: "8080"
  allowTesting: {{if eq .Values.env "staging"}}"true"{{else}}"false"{{end}}
```

Do a quick test using a variable override.

```bash
# With env definition
helm template first-chart . --set env=staging --show-only templates/configmap.yaml
---
# Source: sample-chart/templates/configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap-0.1.0
data:
  port: "8080"
  allowTesting: "true"

# Without env definition
helm template first-chart . --show-only templates/configmap.yaml
---
# Source: sample-chart/templates/configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: first-chart-configmap-0.1.0
data:
  port: "8080"
  allowTesting: "false"
```


## Tips

### Hyper-V vSwitch Configuration

Default Hyper-V network configuration does not permit traffic between existing vSwitches. This is an issue when you need to talk to for example minikube instance running on Hyper-V connected to `Default Switch` from a WSL2 instance connected to `WSL` vSwitch.

In order to permit this traffic, execute the following command from Powershell prompt:

```powershell
Get-NetIPInterface | where {$_.InterfaceAlias -eq 'vEthernet (WSL)' -or $_.InterfaceAlias -eq 'vEthernet (Default Switch)'} | Set-NetIPInterface -Forwarding Enabled
```

Or add the following line to your shell rc file.

```bash
powershell.exe -c "Get-NetIPInterface | where {\$_.InterfaceAlias -eq 'vEthernet (WSL)' -or \$_.InterfaceAlias -eq 'vEthernet (Default Switch)'} | Set-NetIPInterface -Forwarding Enabled 2> \$null"
```

### Minikube Docker Configuration

Docker Desktop's `default` context configuration has `Host` set to `unix:///var/run/docker.sock` when using WSL2 integration.

In order to interact with docker daemon running on minikube instance you can update environment variables. For example:

```bash
export DOCKER_TLS_VERIFY=1
export DOCKER_HOST=tcp://$(minikube ip):2376
export DOCKER_CERT_PATH=/mnt/c/Users/maros_kukan/.minikube
```

This will affect the current shell session only. To make changes persistent you can create a new context.

Retrieve the Minikube's host address and certificate paths:

```bash
DOCKER_HOST="tcp://$(minikube ip):2376"
DOCKER_CERT_PATH=$(wslpath "$(wslvar USERPROFILE)/.minikube/certs")
```

Create and update Minikube context:

```bash
docker context create minikube &>/dev/null
--description "Secured connection from localhost toward minikube" \
--docker "host=${DOCKER_HOST},ca=${DOCKER_CERT_PATH}/ca.pem,cert=${DOCKER_CERT_PATH}/cert.pem,key=${DOCKER_CERT_PATH}/key.pem" &>/dev/null
```

Switch to context

```bash
docker context use minikube
```

Verify control plane:

```bash
docker info -f {{.Name}}
```

```bash
minikube
```

To switch back to default context

```bash
docker context use default
```

```bash
docker info -f {{.Name}}
```

```bash
docker-desktop
```

### Minikube Tunnel Configuration

With `minikube tunnel` it might happen that existing the foreground process in Windows does not remove the route that was added after process closure. This can be observerd by looking at host's active routes.

Show all routes that include minikube's address:

```powershell
netsh int ipv4 sh route | FindStr $(minikube ip)
```

```powershell
No       Manual    1    10.96.0.0/12               39  172.17.237.195
```

To remove them manually:

```powershell
netsh int ipv4 delete route 10.96.0.0/12 39 $(minikube ip)
```

### Play with K8s

Reference deployment settings for play with k8s labs.

Within the web console use `CTRL+Insert` for copy and `SHIFT+Insert` for paste.

On the first node, initialize cluster master node, cluster networking and create deployment.
```bash

kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16

kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml

kube apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/nginx-app.yaml
```

On the second and third worker node, join cluster using the output from `kubeadm init` command.

```bash
kubeadm join 192.168.0.8:6443 \
--token abcdef.0123456789abcdef \
--discovery-token-ca-cert-has sha256
```

Finally verify the cluster status.

```
kubectl cluster-info
kubectl get-nodes -o wide
```

### Kubectl Configuration

```bash
export KUBECONFIG=~/.kube/<yourconfiguration-file>.yaml
```
