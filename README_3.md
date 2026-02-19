## MongoDB as service on Multi Node vCluster deployed inside Docker with local PV/PVC.

See `vcluster_3.yaml`

1. Create Cluster

```bash
# Create a vCluster in Docker (automatically connects)
vcluster create my-mongo --values vcluster_3.yaml

# Verify it's working
kubectl get nodes
kubectl get namespaces
```

2. Create Namesapce

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: db
EOF
```

3. Create PV and PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-vc3-pv-mongo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mongo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc-claim
  namespace: db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: my-vc3-pv-mongo
  storageClassName: ""
EOF
```

or

```bash
kubectl apply -f 1.mongodb-pvclaim.yaml
```

**Confirm created:**

```bash
kubectl get pv -o wide
```

NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE   VOLUMEMODE
my-vc3-pv-mongo   1Gi        RWO            Retain           Bound    db/mongo-pvc-claim                  <unset>                          13s   Filesystem


```bash
kubectl get pvc -n db -o wide
```

NAME              STATUS   VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE   VOLUMEMODE
mongo-pvc-claim   Bound    my-vc3-pv-mongo   1Gi        RWO                           <unset>                 41s   Filesystem

For more detail:

```bash
kubectl describe pvc mongo-pvc-claim -n db
```

Name:          mongo-pvc-claim
Namespace:     db
StorageClass:  
Status:        Bound
Volume:        my-vc3-pv-mongo
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       <none>
Events:        <none>

4. Create Secrets

```bash
kubectl apply -f mongo/2.mongodb-secrets.yaml
```

**Confirm created:**

```bash
kubectl get secrets -n db
```

NAME          TYPE     DATA   AGE
mongo-creds   Opaque   2      4m8s

5. Deploy MongoDB

```bash
kubectl apply -f mongo/3.mongodb-deployment.yaml
```

**Confirm created:**

```bash
kubectl get all -n db -o wide
```

6. Deploy Nodeport Service

```bash
kubectl apply -f mongo/4.mongodb-nodeport-svc.yaml
```

7. Verify deployment, service and storage

```bash
kubectl get all -n db -o wide
# or for more information
kubectl describe pod/mongo-675f47df54-99np2 -n db
```

**Check Connectivity from HOST**

```bash
kubectl port-forward service/mongo-nodeport-svc 27017:27017 -n db
```

Then download and install and use: [MongoDB Compass](https://www.mongodb.com/try/download/compass)

OR

```bash
kubectl exec pod/mongo-675f47df54-99np2 -n db -- /bin/bash
mongosh --host mongo-nodeport-svc --port 27017 -u adminuser -p password123

show dbs
# Returns
#admin   100.00 KiB
#config   60.00 KiB
#local    64.00 KiB

show collections

use admin
# Returns
#switched to db admin

show collections
# Returns
#system.users
#system.version

```

### References

[mongodb-shell-commands](https://www.tutorialsteacher.com/mongodb/mongodb-shell-commands)
[mongo-shell-basic-commands](https://www.bmc.com/blogs/mongo-shell-basic-commands/)
[how-to-use-the-mongodb-shell](https://www.digitalocean.com/community/tutorials/how-to-use-the-mongodb-shell)

[MongoDB Compass](https://www.mongodb.com/try/download/compass)
