# Certified Kubernetes Administrator Exam Tips

## Requirements

- Minikube
- kubectl

## Setup

Create a three node cluster with the following addons.

```bash
# Create new cluster
minikube start --addons=ingress,ingress-dns,metrics-server \
               --cni=cilium \
               --container-runtime=containerd \
               --nodes=3

# Verify cluster nodes
kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
minikube       Ready    control-plane   4m1s    v1.31.0
minikube-m02   Ready    <none>          2m13s   v1.31.0
minikube-m03   Ready    <none>          2m6s    v1.31.0

# Create sample namespaces
kubectl apply -f namespaces.yaml
```

Nice to haves:

```bash
alias k='kubectl'
alias kns='kubectl config set-context --current --namespace'
```

## Cleanup

```bash
minikube delete
```