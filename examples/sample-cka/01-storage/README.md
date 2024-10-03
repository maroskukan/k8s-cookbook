# Storage

## Task #1

- Create a PersistentVolume named "mypv" with 10Gi of capacity.
- It should use the "manual" storage class, "/data" as the host path, and the RWO access mode

### Resources

- [Create a PersitentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume)

### Solution

```bash
# Create manifest for Persistent Volume
cat << EOF > mypv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
EOF

# Apply the manifest
k apply -f mypv.yaml

# Verify Persistent Volume
Name:            mypv
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual
Status:          Bound
Claim:           blue/mypvc
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        10Gi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /data
    HostPathType:  
Events:            <none>
```

## Task #2

- Create a new PersitentVolumeClaim named "mypvc" in the blue namespace.
- The PVC should request 1Gi of capacity from the PersistanVolume named "mypv"

### Resources

- [Create a PersistentVolumeClaim](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolumeclaim)

### Solution

```bash
# Create manifest for Persistent Volume Claim
cat << EOF > mypvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: blue
spec:
  storageClassName: manual
  volumeName: mypv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Apply the manifest
k apply -f mypvc.yaml

# Verify the Persitent Volume Claim
k describe pvc mypvc
Name:          mypvc
Namespace:     blue
StorageClass:  manual
Status:        Bound
Volume:        mypv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      10Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       <none>
Events:        <none>
```

## Task #3

- Create a new Pod named "mypvpod" in the blue namespace
- The Pod should use the "nginx" container image
- The container should use the PersistentVolumeClaim named "mypvc" through the "/data" mount path

### Resources

### Solution

```bash
# Create manifest for Pod
cat << EOF > mypvpod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypvpod
  namespace: blue
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: "/data"
      name: mymnt
  volumes:
  - name: mymnt
    persistentVolumeClaim:
      claimName: mypvc
EOF

# Apply the manifest
k apply -f mypvpod.yaml

# Verify the volumeMounts
k get pod mypvpod -o jsonpath='{.spec.containers[*].volumeMounts[0]}' | jq
{
  "mountPath": "/data",
  "name": "mymnt"
}
```