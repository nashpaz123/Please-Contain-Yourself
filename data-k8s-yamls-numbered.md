## 1) Pod (gitea) — page 35
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

---

## 2) ReplicaSet (nginx) — page 41
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

---

## 3) Deployment (nginx) — page 47
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

---

## 4) Deployment (pg) — stateless — page 56
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

---

## 5) PersistentVolumeClaim (PVC) — page 62
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

---

## 6) StorageClass — page 63
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: my-sc
provisioner: kubernetes.io/no-provisioner
parameters:
  type: local
```

---

## 7) PersistentVolume (PV) — page 64
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

---

## 8) Deployment (pg) — with PVC mounted — page 66
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

---

## 9) StatefulSet (pg) — with volumeClaimTemplates — page 72
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
