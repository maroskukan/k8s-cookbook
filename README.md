# Kubernetes

- [Kubernetes](#kubernetes)
  - [Deployments](#deployments)
    - [Setup](#setup)
    - [Scaling](#scaling)
    - [Cleanup](#cleanup)
  - [Tips](#tips)
    - [Hyper-V vSwitch Configuration](#hyper-v-vswitch-configuration)
    - [Minikube Docker Configuration](#minikube-docker-configuration)

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
# Create context
docker context create minikube \
--description "Secured connection from localhost toward minikube" \
--docker "host=$DOCKER_HOST,ca=$DOCKER_CERT_PATH/ca.pem,cert=$DOCKER_CERT_PATH/cert.pem,key=$DOCKER_CERT_PATH/key.pem"

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