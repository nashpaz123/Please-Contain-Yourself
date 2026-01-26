https://drive.google.com/file/d/1TnEAr9yTFhvlw5HVb0aS6X-Zv8FuwoPj/view?usp=sharing

## 1) Pod (gitea) — page 35
`vim gitea.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitea-pod
spec:
  containers:
    - name: gitea-container
      image: gitea/gitea:1.4
```
`kubectl apply -f gitea.yaml`

`kubectl get pod,rs,deploy`

---

## 1b) expose pod via service - page 27
`vim clientserver.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-server
  labels:
    app: nginx-server
spec:
  containers:
    - name: nginx-server
      image: nginx
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
    - name: client
      image: nginx
```
`kubectl apply -f clientserver.yaml`

`kubectl expose pod nginx-server --port=80 --target-port=80`

`kubectl get pod,svc`

`kubectl exec client<name of any pod in the cluster> -- curl nginx-server`

---

## 2) ReplicaSet (nginx) — page 41
`vim rs.yaml`

```yaml
kind: ReplicaSet
apiVersion: apps/v1
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```
`kubectl apply -f rs.yaml`

---

## 3) Deployment (nginx) — page 47
`vim deploy.yaml`

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```
`kubectl apply -f deploy.yaml`

---

## 4) Deployment (pg) — stateless — page 56
`vim less.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pg
  template:
    metadata:
      labels:
        app: pg
    spec:
      containers:
        - name: postgresql
          image: postgres:14
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
```
`kubectl apply -f less.yaml`

---

## 5) StorageClass — page 63
`vim sc.yaml`

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: my-sc
provisioner: kubernetes.io/no-provisioner
parameters:
  type: local
```
`kubectl apply -f sc.yaml`

`kubectl get sc,pv,pvc`

---

## 6) PersistentVolume (PV) — page 64
`vim pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  storageClassName: my-sc
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres
```
`kubectl apply -f pv.yaml`

---

## 7) PersistentVolumeClaim (PVC) — page 62
`vim claim.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: my-sc
  volumeName: my-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
`kubectl apply -f claim.yaml`

---

## 8) Deployment (pg) — with PVC mounted — page 66
`vim pgdeploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pg
  template:
    metadata:
      labels:
        app: pg
    spec:
      containers:
        - name: postgres
          image: postgres:14
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: pgdata
          persistentVolumeClaim:
            claimName: my-pvc
```
`kubectl apply -f pgdeploy.yaml`

---

## 9) StatefulSet (pg) — with volumeClaimTemplates — page 72
`kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml`

`vim stateful.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pg
  template:
    metadata:
      labels:
        app: pg
    spec:
      containers:
        - name: postgres
          image: postgres:14
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
          volumeMounts:
            - name: pvc
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: pvc
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "local-path"
        resources:
          requests:
            storage: 1Gi
```
`kubectl apply -f stateful.yaml`

`kubectl get sts,pods`

---

## 10) animals
`vim animals.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: animals-svc
  labels:
    app: animals-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    name: http
  selector:
    app: animals-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: animals-app
  labels:
    app: animals-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: animals-app
  template:
    metadata:
      labels:
        app: animals-app
    spec:
      containers:
      - name: animals-app
        image: supergiantkir/animals:bear
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```
`kubectl apply -f animals.yaml`

`kubectl get svc,pod,deploy`


### Go to the external service (load balancer) and visit the zoo, then change the version:

`kubectl set image deployments/animals-app animals-app=supergiantkir/animals:moose`
