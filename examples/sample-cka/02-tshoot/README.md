# Troubleshooting

## Evaluate Cluster and logging

- kubectl cluster-info dump
- kubectl get events
- kubectl top
- /var/log

## Task #1

- Identify the Pod that consumes the highest amount of CPU resources in the red namespace
- Identify the Node that consumes the highest amouont of memory resources
- Execute kubectl cluster-info dump to see detailed information about the cluster

### Solution

```bash
# Create manifest for sample deployment
cat << EOF > mydeployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mydeployment
  name: mydeployment
  namespace: red
spec:
  replicas: 10
  selector:
    matchLabels:
      app: mydeployment
  template:
    metadata:
      labels:
        app: mydeployment
    spec:
      containers:
      - image: nginx
        name: nginx
EOF

# Apply the manifest
k apply -f mydeployment.yaml

# Verify Pod Resource Utilization
k top pod
NAME                            CPU(cores)   MEMORY(bytes)
mydeployment-6f6b5c484d-26bgg   0m           7Mi
mydeployment-6f6b5c484d-57zmp   0m           7Mi
mydeployment-6f6b5c484d-6sf9c   3m           7Mi
mydeployment-6f6b5c484d-cl996   1m           7Mi
mydeployment-6f6b5c484d-hghkr   1m           7Mi
mydeployment-6f6b5c484d-nbn69   2m           7Mi
mydeployment-6f6b5c484d-svzfw   1m           7Mi
mydeployment-6f6b5c484d-tglk7   0m           7Mi
mydeployment-6f6b5c484d-vhzk8   2m           7Mi
mydeployment-6f6b5c484d-zlt7d   2m           7Mi

# Verify Node Resource Utilization
k top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube       323m         4%     1112Mi          3%
minikube-m02   57m          0%     455Mi           1%
minikube-m03   157m         1%     366Mi           1%

# Export cluster-info dump
k cluster-info dump -o json
```

## Task #2

- Inspect the logs in the Pod loated in the red namespace

### Solution

```bash
# Scale down the existing deployment to 1 replica
k scale deployment mydeployment -n red --replicas=1

# Verify the resources in red namespace
k get all -n red
NAME                                READY   STATUS    RESTARTS   AGE
pod/mydeployment-6f6b5c484d-26bgg   1/1     Running   0          7m34s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mydeployment   1/1     1            1           7m34s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/mydeployment-6f6b5c484d   1         1         1       7m34s

# Verify logs from the existing pod
k logs pod/mydeployment-6f6b5c484d-26bgg | head -3
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
```

## Task #3

- Fix the failing deployment in the red namespace

### Introduce an error

```bash
# Introduce na error
k set image deployment/mydeployment -n red nginx=nginx:latestt

# Delete all currently running containers
k delete pods -l app=mydeployment -n red --field-selector status.phase=Running

# Scale out
k scale deployment mydeployment -n red --replicas=5
```

### Solution

```bash
# Get all pods
k get pod
NAME                            READY   STATUS             RESTARTS   AGE
mydeployment-766fbfd77c-9pzwl   0/1     ErrImagePull       0          9s
mydeployment-766fbfd77c-cxkch   0/1     ErrImagePull       0          9s
mydeployment-766fbfd77c-df8r6   0/1     ImagePullBackOff   0          68s
mydeployment-766fbfd77c-jwzxw   0/1     ErrImagePull       0          68s
mydeployment-766fbfd77c-t6shv   0/1     ImagePullBackOff   0          3m37s

# Describe
k describe mydeployment-766fbfd77c-9pzwl | grep image
  Normal   BackOff    24s (x2 over 47s)  kubelet            Back-off pulling image "nginx:latestt"
  Normal   Pulling    12s (x3 over 48s)  kubelet            Pulling image "nginx:latestt"
  Warning  Failed     11s (x3 over 48s)  kubelet            Failed to pull image "nginx:latestt": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:latestt": failed to resolve reference "docker.io/library/nginx:latestt": docker.io/library/nginx:latestt: not found

# Fix
k set image deployment/mydeployment -n red nginx=nginx:latest

# Verify
k get pod
NAME                            READY   STATUS        RESTARTS   AGE
mydeployment-69c4875565-cc2mp   1/1     Running       0          2s
mydeployment-69c4875565-cnt6h   1/1     Running       0          2s
mydeployment-69c4875565-frdz5   1/1     Running       0          4s
mydeployment-69c4875565-rxjqt   1/1     Running       0          4s
mydeployment-69c4875565-vgknw   1/1     Running       0          4s
mydeployment-766fbfd77c-t6shv   0/1     Terminating   0          4m57s
```

## Task #4

- Detect the nonfunctional node within the luster and resolve the issue

### Introduce an error

```bash
minikube ssh -n='minikube-m02'
sudo systemctl stop kubelet
```


### Solution

```bash
# Verify node status
k get nodes
NAME           STATUS     ROLES           AGE     VERSION
minikube       Ready      control-plane   3h24m   v1.31.0
minikube-m02   NotReady   <none>          3h22m   v1.31.0
minikube-m03   Ready      <none>          3h22m   v1.31.0

# Describe faulty node
k get node minikube-m02 -o json | jq '.status.conditions[-1]'
{
  "lastHeartbeatTime": "2024-10-03T11:27:10Z",
  "lastTransitionTime": "2024-10-03T11:29:18Z",
  "message": "Kubelet stopped posting node status.",
  "reason": "NodeStatusUnknown",
  "status": "Unknown",
  "type": "Ready"
}

# Fix the node

```bash
minikube ssh -n='minikube-m02'
sudo systemctl restart kubelet
```

## Task #5

- Detect the nonfunctional node within the luster and resolve the issue

### Introduce an error

```bash
k expose deployment mydeployment --name=myservice --type=ClusterIP --port=80 --target-port=8080 -n red

k patch service myservice -p '{"spec": {"selector": {"app": "mydeploymentt"}}}'
```

### Solution

There are no endpoint objects matching the Selector value.

```bash
# Verify the service status
k describe svc/myservice
Name:                     myservice
Namespace:                red
Labels:                   app=mydeployment
Annotations:              <none>
Selector:                 app=mydeploymentt
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.70.14
IPs:                      10.110.70.14
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
Endpoints:                
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>

# Apply fix
k patch service myservice -p '{"spec": {"selector": {"app": "mydeployment"}}}'

# Verify the Endpoints
k get endpoints myservice
NAME        ENDPOINTS                                                        AGE
myservice   10.244.0.15:8080,10.244.0.41:8080,10.244.2.15:8080 + 2 more...   10m
```