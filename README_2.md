## Multi Node vCluster, on Docker with local PV/PVC.

See `vcluster_2.yaml`

1. Create Cluster

```bash
# Create a vCluster in Docker (automatically connects)
vcluster create my-vc2 --values vcluster_2.yaml

# Verify it's working
kubectl get nodes
kubectl get namespaces
```

2. Install [Traefik](https://traefik.io/traefik) Ingress controller

```bash
helm upgrade --install traefik traefik \
  --repo https://helm.traefik.io/traefik \
  --namespace ingress-traefik --create-namespace
```

3. Wait for Traefik pod to be ready

```bash
kubectl get pods -n ingress-traefik -w -o wide
```

4. Create PV and PVC

```bash
# PV's are not tied to a namespace.
# PVC's are tied to a namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-vc-pv-nginx
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/nginx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-vc-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: my-vc-pv-nginx
  storageClassName: ""
EOF
```

5. Deploy nGinx pod, service and create ingress

```bash
# App my-vc -> Ingress container
# With PVC -> my-vc-data
cat <<EOF | kubectl apply -f -
 apiVersion: apps/v1
 kind: Deployment
 metadata:
  name: my-vc
  namespace: default
 spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-vc
  template:
    metadata:
      labels:
        app: my-vc
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: my-vc-data
---
apiVersion: v1
kind: Service
metadata:
  name: my-vc-svc
  namespace: default
spec:
  selector:
    app: my-vc
  ports:
    - port: 80
      targetPort: 80
EOF
```

6. Deploy Ingress tied to our [Traefik](https://traefik.io/traefik) service
   
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-vc-ingress
  namespace: default
spec:
  ingressClassName: traefik
  rules:
    - host: my-vc.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-vc-svc
                port:
                  number: 80
EOF
```

7. Add a test index.html

```bash
kubectl exec -it $(kubectl get pod -l app=my-vc -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "<h1>vCluster + Traefik works!</h1>" > /usr/share/nginx/html/index.html'
```


8. Get the NodePort

```bash
kubectl get svc -n ingress-traefik
```

9. Get the container IP

```bash
docker inspect vcluster.cp.my-vc2 | grep IPAddress
```

10. Test it (replace with your ip and nodePort from step 6, use the http port, not https)

```bash
docker exec vcluster.cp.my-vc2 curl -s -H "Host: my-vc.local" http://<container_ip>:<nodeport>
# docker exec vcluster.cp.my-vc2 curl -s -H "Host: my-vc.local" http://172.19.0.2:32576

```

docker exec vcluster.cp.my-vc2 curl -s -H "Host: my-vc.local" http://172.19.0.2:32346

11.   Verify ingress and storage

```bash
kubectl get ingress,pvc,svc,pods
```
