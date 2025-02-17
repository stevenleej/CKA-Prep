
# Storage in Kubernetes

## Volumes
- Containers are meant to be transient. Data within the container is destroyed when a container terminates.
- Like in Docker, pods are transient. We must attach them to a pod (data persists even when the pod terminates)

Example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example-linux
spec:
  os: { name: linux }
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: example-container
    image: registry.k8s.io/test-webserver
    volumeMounts:
    - mountPath: /foo
      name: example-volume
      readOnly: true
  volumes:
  - name: example-volume
    # mount /data/foo, but only if that directory already exists
    hostPath:
      path: /data/foo # directory location on host
      type: Directory # this field is optional
```

### Persistent Volumes
Persistent volumes allows us to modularize the volume information for a pod as an independent object we can bind to a pod, and simplifies management of storage.
- Users select which PV to use via a PVC

persistent volume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath: <-- can define the host path 
	path: <some host path> 
  persistentVolumeReclaimPolicy: Recycle <-- define reclaim policy once deleted
  storageClassName: slow
  mountOptions: <-- perhaps definition differs per cloud providers
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

### Persistent Volume Claim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

Note: if there are multiple matches for a single claim, you can use label selectors to match with a specific claim. But, if all the criteria matches, the volume claim might be bound to another larger volume and no option is left.

If there are no volumes available, PVC will be left on pending until it is bound to a PV




Example of PVC for a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```


### Storage Classes
Allows us to describe a class of storage for a volume. (in the case of cloud providers, it's like defining a PV as being an EBS volume from AWS)

### Dynamic Provisioning
- We define a provisioner ahead and have it attach to a PV atuomatically. 

### Static Provisiong
- Provisioning the volume in your cloud provider or in your bare-metal manually before assigning it to the PV