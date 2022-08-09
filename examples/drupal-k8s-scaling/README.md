# Scaling Drupal in K8s

## Prerequisites

- NFS Server

## Deployment

```bash
# Add helm repository for nfs-subdir-external-provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# Deploy nfs-subdir-external provisioner in default namespace
NFS_PRIVATE_IP=192.168.173.1

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=$NFS_PRIVATE_IP \
    --set nfs.path=/home/nfs

# Verify the pod status
kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
nfs-subdir-external-provisioner-65b9f476bc-fdmn9   1/1     Running   0          3m14s

# Display storage classes
kubectl get storageclass nfs-client
NAME         PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   3m49s

# Deploy mariadb and drupal definitions
kubectl apply -f drupal.yml -f mariadb.yml
namespace/drupal created
configmap/drupal-config created
persistentvolumeclaim/drupal-files-pvc created
deployment.apps/drupal created
service/drupal created
persistentvolumeclaim/mariadb-pvc created
deployment.apps/mariadb created
service/mariadb created

# Verify the deployment status
kubectl get deployment -n drupal
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
drupal    1/1     1            1           62s
mariadb   1/1     1            1           61

# Retrieve the node port
NODE_PORT=$(kubectl get service drupal -n drupal -o json | jq -r '.spec.ports[].nodePort')

# Retrieve the node address
NODE_ADDRESS=$(kubectl get nodes -o json | jq -r '.items[0].status.addresses[] | select(.type=="ExternalIP").address')

# Complete the drupal installation from web ui
echo "http://$NODE_ADDRESS:$NODE_PORT/"

# Verify the content of /home/nfs on NFS server
ls /home/nfs/
drupal-drupal-files-pvc-pvc-b1e5f282-5d58-4b43-a62e-23c720d69ccb
```


## Testing

```bash
# Get baseline performance using apache benchmark
docker run --rm jordi/ab -n 500 -c 10 http://$NODE_ADDRESS:$NODE_PORT/
...
[ Output omitted for brevity ]
...
Requests per second:    30.56 [#/sec] (mean)
Time per request:       327.231 [ms] (mean)
Time per request:       32.723 [ms] (mean, across all concurrent requests)
Transfer rate:          498.71 [Kbytes/sec] received
...
[ Output omitted for brevity ]
...

# Scale out deployment
kubectl scale --replicas=5 deployment drupal -n drupal

# Get updated performance using apache benchmark
docker run --rm jordi/ab -n 500 -c 10 http://$NODE_ADDRESS:$NODE_PORT/
...
[ Output omitted for brevity ]
...
Requests per second:    49.38 [#/sec] (mean)
Time per request:       340.348 [ms] (mean)
Time per request:       34.035 [ms] (mean, across all concurrent requests)
Transfer rate:          479.49 [Kbytes/sec] received
...
[ Output omitted for brevity ]
...

# Scale in deployment
kubectl scale --replicas=1 deployment drupal -n drupal
```