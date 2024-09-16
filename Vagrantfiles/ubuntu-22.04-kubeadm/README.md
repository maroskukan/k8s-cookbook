# K8S environment

The `Vagrantfile` contains definition for two Ubuntu 22.04 virtual machines - one for control-plane and second to be used for worker node.

## Create

```bash
# Validate the Vagrantfile

# Start and provision the environment
vagrant up --provider=libvirt

# Connect to machine via SSH
vagrant ssh [ cp | worker ]
```

## Cleanup

```bash
# Stop and delete the environment
vagrant destroy --force
```

## Kubernetes Cluster Deployment

### Requirements

The minimum system requirements are
- 2 CPU
- 2 GB of RAM
- Swap disabled
- Full network connectivity between hosts

Swap needs to be disabled for kubelet service to start.

```bash
# Disable swap
sudo swapoff -a

# Disable swap mounting
sudo sed -i '/swap/s/^/#/' /etc/fstab

# Remove swap file (optional)
sudo rm /swap.img
```

Install common dependencies.

```bash
# # Suppress interactive prompts (optional)
export DEBIAN_FRONTEND=noninteractive
export NEEDRESTART_MODE=a

# Skip kernel and firmware update (optional)
# Use this to save some time for Vagrant VM
sudo apt-mark hold linux-image-generic \
                   linux-headers-generic \
                   linux-image-$(uname -r) \
                   linux-headers-$(uname -r) \
                   linux-firmware

# Update package list and upgrade all packages
sudo apt update && sudo apt upgrade -y

# Install common dependencies
sudo apt install -y \
     socat \
     apt-transport-https \
     curl \
     ca-certificates
```

Enable packet forwarding.

```bash
# Enable packet forwarding
sudo tee /etc/sysctl.d/10-k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply new settings
sudo sysctl --system
```

Create host record for control plane.

```bash
# Create new host record
ip=$(ip addr show eth0 | grep 'inet ' | awk '{print $2}' | cut -d'/' -f1)
echo $ip k8s-cp | sudo tee -a /etc/hosts

# Verify
Server:         127.0.0.53
Address:        127.0.0.53#53

Name:   k8s-cp
Address: 192.168.121.37
```


### Container runtime

Install and configure container runtime, ensure that the correct [cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers) is set.

```bash
# Load the modules on startup
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load the modules now
sudo modprobe overlay
sudo modprobe br_netfilter

# Install container runtime
sudo apt install -y containerd

# Configure container runtime
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml

# Restart container runtime
sudo systemctl restart containerd

# Verify container runtime
systemctl status containerd
```

Verify container creation

```bash
# Pull down the image
sudo ctr image pull docker.io/library/hello-world:latest

# Start the container
sudo ctr run --rm --tty docker.io/library/hello-world:latest hello-world

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

# Remove the image
sudo ctr images rm docker.io/library/hello-world:latest
```

### Kubernetes binaries


```bash
# Download the public signing key for the Kubernetes package repositories
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add specific version repository /etc/apt/sources.list.d/kubernetes.list
sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
EOF

# Update the package index
sudo apt update

# Install kubernetes binaries
sudo apt install -y kubeadm kubelet kubectl

# Pin kubernetes binaries version
sudo apt-mark hold kubeadm kubelet kubectl
```

Verify

```bash
# Verify
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.5", GitCommit:"74e84a90c725047b1328ff3d589fedb1cb7a120e", GitTreeState:"clean", BuildDate:"2024-09-12T00:17:07Z", GoVersion:"go1.22.6", Compiler:"gc", Platform:"linux/amd64"}


kubelet --version
Kubernetes v1.30.5

kubectl version --client
Client Version: v1.30.5
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### Cluster configuration

Create configuration file for the cluster. The controlPlaneEndpoint is the IP address on CP primary network interface.

```bash
tee kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.30.5
controlPlaneEndpoint: "k8s-cp:6443"
networking:
  podSubnet: 10.244.0.0/16
EOF
```

### Cluster initiliazation

```bash
# Dry run
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs --dry-run

# Initialize the cluster - save the output
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
```

Verify the cluster current state

```bash
# Prepare kubeconfig for current user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify cluster info
kubectl cluster-info

# Output
Kubernetes control plane is running at https://192.168.121.178:6443
CoreDNS is running at https://192.168.121.178:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Verify the node state
NAME   STATUS     ROLES           AGE     VERSION
cp     NotReady   control-plane   3m19s   v1.30.5
```

As you can see the `cp` is `NotRead`. To investigate why can use ask for more details. We can filter the output using `jq` for easier reading.

```bash
sudo apt install -y jq
```

```bash
kubectl get node cp -o json | jq '.status.conditions[] | select(.type == "Ready")'
```

```json
{
  "lastHeartbeatTime": "2024-09-16T08:48:46Z",
  "lastTransitionTime": "2024-09-16T08:38:18Z",
  "message": "container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized",
  "reason": "KubeletNotReady",
  "status": "False",
  "type": "Ready"
}
```

As you can see, the node is not ready because we are missing the Network Plugin. Lets deploy one on next step.

### Container Network Interface plugin

Calico is a high-performance networking and network security solution for Kubernetes that provides scalable and flexible networking and network policies.

```bash
# Install the tigera operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml

# Download the custom resource definition
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml -O

# Modify the manifest's cidr value
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/' custom-resources.yaml

# Apply the manifest to install Callico
kubectl create -f custom-resources.yaml

# Verify Calico Installation
watch kubectl get pods -n calico-system

# Output
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-fdc4f945d-tlqvr   1/1     Running   0          65s
calico-node-7pmfb                         1/1     Running   0          66s
calico-typha-7d8d9c4bd4-vxs2b             1/1     Running   0          66s
csi-node-driver-4rf4d                     2/2     Running   0          65s
```

Verify node and pod state.

```bash
kubectl get nodes
NAME   STATUS   ROLES           AGE   VERSION
cp     Ready    control-plane   15m   v1.30.5

kubectl get pods -n kube-system
NAME                         READY   STATUS    RESTARTS   AGE
coredns-55cb58b774-phgps     1/1     Running   0          15m
coredns-55cb58b774-r49c7     1/1     Running   0          15m
etcd-cp                      1/1     Running   0          15m
kube-apiserver-cp            1/1     Running   0          15m
kube-controller-manager-cp   1/1     Running   0          15m
kube-proxy-f6dj5             1/1     Running   0          15m
kube-scheduler-cp            1/1     Running   0          15m
```

### Test Pod creation

```bash
# Create a simple nginx pod
kubectl run nginx --image=nginx --port=80

# Verify that pod events
kubectl get pod nginx -o json | jq -r '.status.conditions'

# Output
[
  {
    "lastProbeTime": null,
    "lastTransitionTime": "2024-09-16T12:01:41Z",
    "message": "0/1 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.",
    "reason": "Unschedulable",
    "status": "False",
    "type": "PodScheduled"
  }
]
```

As you can see the pod scheduling failed, the reason is that by default the control plane node is not allowed to schedule production workloads.

To solve this we need to remove the default `NoSchedule` taint that is applied to cp node.

```bash
# Display existing taings
kubectl get node cp -o json | jq -r '.spec.taints'

# Output
[
  {
    "effect": "NoSchedule",
    "key": "node-role.kubernetes.io/control-plane"
  }
]
```

Remove this taint

```bash
kubectl taint nodes cp node-role.kubernetes.io/control-plane-

# Output
node/cp untained
```

Verify the nginx pod status

```bash
kubectl get pod nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          7m51s
```

Access the nginx pod,

```bash
# Port forward to localhost
kubectl port-forward pod/nginx 8080:80 &

# Test the application
curl -I http://localhost:8080

# Output
Handling connection for 8080
HTTP/1.1 200 OK
Server: nginx/1.27.1
Date: Mon, 16 Sep 2024 12:11:48 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 12 Aug 2024 14:21:01 GMT
Connection: keep-alive
ETag: "66ba1a4d-267"
Accept-Ranges: bytes
```

Stop the port-worward and remove the pod.

```bash
kubectl delete pod nginx
```

The control plane is now ready. In order to join worker nodes you need to retrieve the token value from the control plane.

```bash
kubeadm token create --print-join-command

# Output
kubeadm join k8s-cp:6443 --token 1hkzn3.scbykz4utq6epfyj --discovery-token-ca-cert-hash sha256:3b4dd0e38c99169c83d1e3d8338c20264f540b36a9a5ea500c2400b8f0d21860
```

### Worker

#### Adding

Follow the same steps to setup worker nodes as for control plane with the following changes:
- Update `/etc/hosts` with correct IP address for `k8s-cp` host
- Installing `kubectl` is not required
- Instead of `kubeadm init` command use the `kubeadm join` command described in last step.


```bash
sudo kubeadm join k8s-cp:6443 --token 1hkzn3.scbykz4utq6epfyj --discovery-token-ca-cert-hash sha256:3b4dd0e38c99169c83d1e3d8338c20264f540b36a9a5ea500c2400b8f0d21860 | tee kubeadm-join.out
```

On `cp` node, verify.

```bash
kubectl get nodes
NAME     STATUS   ROLES           AGE   VERSION
cp       Ready    control-plane   51m   v1.30.5
worker   Ready    <none>          84s   v1.30.5
```

#### Removing

```bash
# Drain the node
kubectl drain worker --ignore-daemonsets --delete-emptydir-data

# Verify the node
NAME      STATUS                     ROLES           AGE   VERSION
cp        Ready                      control-plane   75m   v1.30.5
worker    Ready,SchedulingDisabled   <none>          25m   v1.30.5
worker2   Ready                      <none>          11m   v1.30.5

# Remove the node
kubectl delete node worker
```

Clean up the node.

```bash
# Stop the kubelet service
sudo systemctl stop kubelet

# Remove kubernetes packages
sudo apt-get remove --purge kubelet kubeadm

# Remove kubernetes configuration
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
```

