# Kubernetes

- [Kubernetes](#kubernetes)
  - [Documentation](#documentation)
  - [Deployments](#deployments)
    - [Setup](#setup)
    - [Scaling](#scaling)
    - [Cleanup](#cleanup)
  - [Tips](#tips)
    - [Hyper-V vSwitch Configuration](#hyper-v-vswitch-configuration)
    - [Minikube Docker Configuration](#minikube-docker-configuration)
    - [Minikube Tunnel Configuration](#minikube-tunnel-configuration)
    - [Play with K8s](#play-with-k8s)
    - [Kubectl Configuration](#kubectl-configuration)


## Documentation

- [Minikube](https://minikube.sigs.k8s.io/)
- [Play With k8s](https://labs.play-with-k8s.com/)

## Deployments

### Setup

```bash
# Create new deployment from hello-go image
kubectl create deployment hello-go

# Check status of deployment
kubectl get deployment hello-go

# Check status of pods
kubectl get pods

# Get more details about pod
kubectl describe pod hello-go

# Modify deployment
kubectl edit deployment hello-go

# Expose deployment
kubectl expose deployment hello-go --type=LoadBalancer --port=8180

# Check the service
kubectl get service hello-go

#Interact with service (when using minikube)
minikube service hello-go

# Check instances logs
kubectl logs -l app=hello-go
```

### Scaling

```bash
# Increase number of instances to 4
kubectl scale deployments/hello-go --replicas=4

# Check deployment
kubectl get deployment hello-go

# Get logs by pod
# https://github.com/kubernetes/kubernetes/pull/76471
for pod in $(kubectl get po -l app=hello-go -oname); do echo $pod; kubectl logs $pod; done;

kubectl logs -lapp=hello-go --all-containers --prefix
```

### Cleanup

```bash
# Delete the Service
kubectl delete service hello-go

# Delete the Deployment and associated pods
kubectl delete deployment hello-go

# Delete the container image
docker rmi hello-go
```


## Tips

### Hyper-V vSwitch Configuration

Default Hyper-V network configuration does not permit traffic between existing vSwitches. This is an issue when you need to talk to for example minikube instance running on Hyper-V connected to `Default Switch` from a WSL2 instance connected to `WSL` vSwitch.

In order to permit this traffic, execute the following command from privileged Powershell prompt:

```powershell
Get-NetIPInterface | where {$_.InterfaceAlias -eq 'vEthernet (WSL)' -or $_.InterfaceAlias -eq 'vEthernet (Default Switch)'} | Set-NetIPInterface -Forwarding Enabled
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

```bash
# Retrieve the Minikube's host address and certificate paths
DOCKER_HOST="tcp://$(minikube ip):2376"
DOCKER_CERT_PATH=$(wslpath "$(wslvar USERPROFILE)/.minikube/certs")

# Create and update Minikube context
docker context create minikube &>/dev/null
--description "Secured connection from localhost toward minikube" \
--docker "host=${DOCKER_HOST},ca=${DOCKER_CERT_PATH}/ca.pem,cert=${DOCKER_CERT_PATH}/cert.pem,key=${DOCKER_CERT_PATH}/key.pem" &>/dev/null

# Switch to context
docker context use minikube

# Verify control plane
docker info -f {{.Name}}
minikube

# To switch back to default context
docker context use default

docker info -f {{.Name}}
docker-desktop
```

### Minikube Tunnel Configuration

With `minikube tunnel` it might happen that existing the foreground process in Windows does not remove the route that was added after process closure. This can be observerd by looking at host's active routes.

```powershell
# Show all routes that include minikube's address
netsh int ipv4 sh route | FindStr $(minikube ip)
No       Manual    1    10.96.0.0/12               39  172.17.237.195
```

To remove them manually.

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
