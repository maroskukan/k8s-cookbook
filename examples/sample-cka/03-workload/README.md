# Deployment

## Task #1

- Create a Deployment named "mydeployment" based on the "nginx" image in the purple namespace

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
  namespace: purple
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mydeployment
  strategy: {}
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

# Verify Deployed resources
NAME                                READY   STATUS    RESTARTS   AGE
pod/mydeployment-6f6b5c484d-j7gz9   1/1     Running   0          15s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mydeployment   1/1     1            1           15s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/mydeployment-6f6b5c484d   1         1         1       15s
```

## Task #2

- Create a new ConfigMap named "myconfigmap" in the purple namespace and add the "exam" key with "CKA" as the value.
- Create a new Pod named "myconfigmappod" that uses the "nginx" image
- Add an environment variable named MY_EXAM_ENV that reads the value of "exam" from "myconfigmap"

### Solution

```bash
# Scale down the existing deployment to 1 replica
k create configmap myconfigmap --from-literal=exam=CKA

# Verify the resources in red namespace
k describe cm myconfigmap
Name:         myconfigmap
Namespace:    purple
Labels:       <none>
Annotations:  <none>

Data
====
exam:
----
CKA


BinaryData
====

Events:  <none>

# Create pod manifest
cat << EOF > myconfigmappod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myconfigmappod
  namespace: purple
spec:
  containers:
  - image: nginx
    name: myconfigmappod
    env:
    - name: MY_EXAM_ENV
      valueFrom:
        configMapKeyRef:
          name: myconfigmap
          key: exam
EOF

# Apply the manifest
kubectl apply -f myconfigmappod.yaml

# Verify the env var in pod
k exec myconfigmappod -- env | grep MY_EXAM_ENV
MY_EXAM_ENV=CKA
```

## Task #3

- Scale the deployment named "mydeployment" in the purple namespace to ten replicas.

### Solution

```bash
# Get current number of replicas for mydeployment
k get replicaset -l app=mydeployment
NAME                      DESIRED   CURRENT   READY   AGE
mydeployment-6f6b5c484d   1         1         1       13m

# Scale deployment to 10 replicas
k scale deployment mydeployment --replicas=10

# Verify rs
k get rs -l app=mydeployment
NAME                      DESIRED   CURRENT   READY   AGE
mydeployment-6f6b5c484d   10        10        10      14m

# Verify pods
k get pods -l app=mydeployment
NAME                            READY   STATUS    RESTARTS   AGE
mydeployment-6f6b5c484d-82jnz   1/1     Running   0          48s
mydeployment-6f6b5c484d-gltj2   1/1     Running   0          48s
mydeployment-6f6b5c484d-grxc2   1/1     Running   0          48s
mydeployment-6f6b5c484d-j652v   1/1     Running   0          48s
mydeployment-6f6b5c484d-j7gz9   1/1     Running   0          15m
mydeployment-6f6b5c484d-m847z   1/1     Running   0          48s
mydeployment-6f6b5c484d-nxwzb   1/1     Running   0          48s
mydeployment-6f6b5c484d-rhckw   1/1     Running   0          48s
mydeployment-6f6b5c484d-sb6wp   1/1     Running   0          48s
mydeployment-6f6b5c484d-vz5np   1/1     Running   0          48s
```

## Task #4

- Create a pod named "mylivenesspod" based on the "nginx" image in the purple namespace
- The container should implement a liveness probe that checks the availability of the HTTP server every five seconds


### Solution

```bash
# Create pod manifest
cat << EOF > mylivenesspod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mylivenesspod
  namespace: purple
spec:
  containers:
  - image: nginx
    name: mylivenesspod
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
EOF
# Verify pod status
k get pod mylivenesspod
NAME            READY   STATUS    RESTARTS   AGE
mylivenesspod   1/1     Running   0          3m21s

# Verify pod livenessProbe
k get pod mylivenesspod -o json | jq '.spec.containers[].livenessProbe'
{
  "failureThreshold": 3,
  "httpGet": {
    "path": "/",
    "port": 80,
    "scheme": "HTTP"
  },
  "periodSeconds": 5,
  "successThreshold": 1,
  "timeoutSeconds": 1
}
```


## Task #5

- Taint the "minikube-m03" Node with the "exam=cka" key-value pair and the NoSchedule effect
- Create a Deployment with 20 replicas named "mydeployment" based on the "nginx" image in the purple namespace.
- Display all the Pods and identify their location
- Modify the Pod spec to tolerata the taint

### Solution

```bash
# Taint the minikube-m03 node
k taint node minikube-m03 exam=cka:NoSchedule

# Verify taint
k get node minikube-m03 -o json | jq '.spec.taints'
[
  {
    "effect": "NoSchedule",
    "key": "exam",
    "value": "cka"
  }
]
# Remove existing deployment
k delete -f mydeployment.yaml

# Create a deployment
k create deployment mydeployment --image=nginx --replicas=20 -n purple

# Verify placement - see... no m03 placement
k get pods -o custom-columns="POD_NAME:.metadata.name,STATE:.status.phase,NODE:.spec.nodeName"
POD_NAME                        STATE     NODE
myconfigmappod                  Running   minikube-m02
mydeployment-6f6b5c484d-2t582   Running   minikube
mydeployment-6f6b5c484d-5dmgd   Running   minikube
mydeployment-6f6b5c484d-5nxwp   Running   minikube
mydeployment-6f6b5c484d-669n7   Running   minikube-m02
mydeployment-6f6b5c484d-78frz   Running   minikube-m02
mydeployment-6f6b5c484d-7bsnv   Running   minikube-m02
mydeployment-6f6b5c484d-7vpbh   Running   minikube-m02
mydeployment-6f6b5c484d-9mxw8   Running   minikube-m02
mydeployment-6f6b5c484d-b8nbd   Running   minikube-m02
mydeployment-6f6b5c484d-dvt4n   Running   minikube
mydeployment-6f6b5c484d-m2m59   Running   minikube
mydeployment-6f6b5c484d-nkpv6   Running   minikube
mydeployment-6f6b5c484d-qstk8   Running   minikube-m02
mydeployment-6f6b5c484d-r92fw   Running   minikube
mydeployment-6f6b5c484d-r96k5   Running   minikube-m02
mydeployment-6f6b5c484d-thrhn   Running   minikube-m02
mydeployment-6f6b5c484d-v7qvr   Running   minikube
mydeployment-6f6b5c484d-v7w7c   Running   minikube
mydeployment-6f6b5c484d-v86x4   Running   minikube-m02
mydeployment-6f6b5c484d-vpnc9   Running   minikube
mylivenesspod                   Running   minikube-m02

# Add toleration
kubectl patch deployment mydeployment -p '{"spec": {"template": {"spec": {"tolerations": [{"key": "exam", "operator": "Equal", "value": "cka", "effect": "NoSchedule"}]}}}}'

# Verify placement - see... m03 placement
k get pods -o custom-columns="POD_NAME:.metadata.name,STATE:.status.phase,NODE:.spec.nodeName"
POD_NAME                       STATE     NODE
myconfigmappod                 Running   minikube-m02
mydeployment-7f7c984b5-2cb97   Running   minikube-m02
mydeployment-7f7c984b5-4hlkf   Running   minikube-m03
mydeployment-7f7c984b5-5drwj   Running   minikube-m03
mydeployment-7f7c984b5-7l8cf   Running   minikube
mydeployment-7f7c984b5-bnw4x   Running   minikube
mydeployment-7f7c984b5-cbq4l   Running   minikube-m02
mydeployment-7f7c984b5-cw6dp   Running   minikube-m03
mydeployment-7f7c984b5-czpzs   Running   minikube-m03
mydeployment-7f7c984b5-h6hzd   Running   minikube-m03
mydeployment-7f7c984b5-hgrzc   Running   minikube-m02
mydeployment-7f7c984b5-k4fkn   Running   minikube
mydeployment-7f7c984b5-l6ks7   Running   minikube
mydeployment-7f7c984b5-mjm2j   Running   minikube-m03
mydeployment-7f7c984b5-nf2sz   Running   minikube
mydeployment-7f7c984b5-pvtvx   Running   minikube-m02
mydeployment-7f7c984b5-q5xpq   Running   minikube
mydeployment-7f7c984b5-rltt9   Running   minikube-m03
mydeployment-7f7c984b5-td5kz   Running   minikube-m02
mydeployment-7f7c984b5-tlfjw   Running   minikube-m02
mydeployment-7f7c984b5-zrkf5   Running   minikube-m02
mylivenesspod                  Running   minikube-m02
```


