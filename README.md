# Kubernetes

- [Kubernetes](#kubernetes)
  - [Documentation](#documentation)
  - [Environment](#environment)
    - [Quick start](#quick-start)
  - [Deployments](#deployments)
    - [Setup](#setup)
    - [Scaling](#scaling)
    - [Cleanup](#cleanup)
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
  - [Tips](#tips)
    - [Hyper-V vSwitch Configuration](#hyper-v-vswitch-configuration)
    - [Minikube Docker Configuration](#minikube-docker-configuration)
    - [Minikube Tunnel Configuration](#minikube-tunnel-configuration)
    - [Play with K8s](#play-with-k8s)
    - [Kubectl Configuration](#kubectl-configuration)


## Documentation

- [Minikube](https://minikube.sigs.k8s.io/)
- [Play With k8s](https://labs.play-with-k8s.com/)


## Environment

### Quick start

Start by creating a new python virtual environment:

```bash
pyenv virtualenv 3.9.6 k8s-cookbook
```

Next, install required pip packages:

```bash
pip install --upgrade pip setuptools
pip install -r requirements-dev.txt
```


## Deployments

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
kubectl exec -it <pod-name> /bin/bash
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
