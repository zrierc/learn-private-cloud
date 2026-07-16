# Kubernetes Volumes & Data

- **Date:** July 15th - 16th, 2026

## Overview

Eksplorasi dan belajar mengelola volumes dan data di Kubernetes cluster

References:

- [Official Docs](https://kubernetes.io/docs/concepts/storage/volumes/)

---

## Volumes

Volume dalam Kubernetes memiliki konsep yang sama dengan Docker Volume, yaitu:

- untuk **persistent storage**, tetapi bagi Pods
- **shared storage** (antara Pods)

Meskipun begitu Kube Volumes memiliki mekanisme yang berbeda. Volume di Kubernetes
memiliki tipe yang berbeda, diantaranya:

- emptyDir: temporary, disimpan di memory hostPath: Di mount ke directory dalam nodes
- Persistent Volume (PV) dan Persistent Volume Claim (PVC): Memiliki backend storage
  tersendiri, sehingga dapat disimpan dimanapun, host, object storage, block storage,
  NFS, etc.
  - PV: Actual storage resources, describes real physical storage details
  - PVC: sebuah mekanisme untuk me-request storage ke PV. Doesn't care WHERE or
    HOW it's provided

Sama seperti Network, Volumes juga membutuhkan plugin yang digunakan untuk backend
storagenya. Plugin ini disebut Container Storage Interface (CSI). Beberapa contohnya:

- FC (fibre channel) storage
- NFS storage
- awsElasticBlockStore
- azure disk
- azure file
- cinder
- gcePersistentDisk
- etc..

Selain itu Volume juga memiliki mekanisme / mode untuk mengakses nya, yaitu:

- `ReadWriteOnce` (RWO) - Read/write by a single node; multiple Pods can share
  it if on the same node.
- `ReadOnlyMany` (ROX) - Read-only by multiple nodes.
- `ReadWriteMany` (RWX) - Read/write by multiple nodes.
- `ReadWriteOncePod` (RWOP) (Kubernetes v1.29, stable) - Read/write by a single
  Pod across the entire cluster.

---

## Persistent Volume (PV)

Merupakan storage resources yang ada di cluster level. Bisa dibuat dengan 2 cara:

- Static provisioning / manual dibuat oleh Kubernetes Administrator
- Dynamic provisioning / otomatis dibuatkan dengan **StorageClass**
- Membutuhkan Backend, connect langsung ke CSI

Setiap PV memiliki:

- Kapasitas
- Access Mode
- Volume Mode: `FileSystem` atau `Block`
- Retain Policy
  - `Retain`: keep data after release
  - `Delete`: delete storage resources after release
- Tidak terikat pada namespace, sehingga dapat di claim oleh PVC di namespace manapun

### List Available PV

```sh
kubectl get persistentvolumes

# atau lebih umum
kubectl get pv
```

### Example PV hostFile

Membuat PV bernama `my-pv` dengan kapasitas `1GB` dan disimpan di `/mnt/data`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

### Example PV NFS

Sebagai pre-requisite, setiap node harus di-install package nfs (misalnya `nfs-common`).
Kemudian mendeploy NFS server, misalnya:

```yaml
apiVersion: v1
items:
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        name: nfs-server
      name: nfs-server
    spec:
      progressDeadlineSeconds: 2147483647
      replicas: 1
      revisionHistoryLimit: 2147483647
      selector:
        matchLabels:
          name: nfs-server
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
        type: RollingUpdate
      template:
        metadata:
          creationTimestamp: null
          labels:
            name: nfs-server
        spec:
          containers:
            - image: registry.adinusa.id/btacademy/volume-nfs:0.8
              imagePullPolicy: IfNotPresent
              name: nfs-server
              ports:
                - containerPort: 2049
                  name: nfs
                  protocol: TCP
                - containerPort: 20048
                  name: mountd
                  protocol: TCP
                - containerPort: 111
                  name: rpcbind
                  protocol: TCP
              resources: {}
              securityContext:
                privileged: true
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
              volumeMounts:
                - mountPath: /exports
                  name: mypvc
          dnsPolicy: ClusterFirst
          nodeSelector:
            kubernetes.io/hostname: pod-username-worker1
          restartPolicy: Always
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
          volumes:
            - hostPath:
                path: /data
                type: ""
              name: mypvc
    status: {}
  - apiVersion: v1
    kind: Service
    metadata:
      creationTimestamp: null
      name: nfs-server
    spec:
      ports:
        - name: nfs
          port: 2049
          protocol: TCP
          targetPort: 2049
        - name: mountd
          port: 20048
          protocol: TCP
          targetPort: 20048
        - name: rpcbind
          port: 111
          protocol: TCP
          targetPort: 111
      selector:
        name: nfs-server
      sessionAffinity: None
      type: ClusterIP
    status:
      loadBalancer: {}
kind: List
metadata: {}
```

> [!TIP]
> Untuk mendapatkan IP untuk field `server` dapat menggunakan perintah: </br>
> `kubectl get svc | grep nfs-server`

Membuat PV bernama `nfs-data` sebesar `1MB` yang terhubung dengan backend NFS dengan
path `/exports`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-data
spec:
  capacity:
    storage: 1Mi
  accessModes:
    - ReadWriteMany
  nfs:
    # FIXME: use the right IP
    server: use IP from nfs-server ClusterIP
    path: "/exports"
```

---

## Persistent Volume Claim (PVC)

> [!IMPORTANT]
> `PVC` tetap dibutuhkan baik itu saat menggunakan `PV` ataupun `StorageClass`
> karena `PVC` adalah mekanisme untuk melakukan request ke storage

Ketika menggunakan Volumes, Pods menggunakan PVC untuk request storage dengan size
dan tipe yang spesifik, kemudian Kubernetes binds Pod tersebut ke PV yang cocok.

> [!NOTE]
> Jika Kubernetes tidak menemukan PV atau StorageClass yang cocok, maka Kubernetes
> akan membuat baru PV yang dinamis

PVC me-request dalam scope namespace. Sebuah PVC hanya bisa di bind ke satu PV
dalam satu waktu.

### List Available PVC

```sh
kubectl get persistentvolumeclaims

# atau lebih umum
kubectl get pvc
```

### Example Create PVC

Membuat PVC bernama `first-pvc` sebesar `8GB` dengan akses Read/Write hanya dalam
satu node.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: first-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

### Example Attach PVC to Pod

Membuat Pod bernama `pod-nginx` dengan image `nginx` kemudian menghubungkan static
content yang ada di `/usr/share/nginx/html` ke PVC yang dibuat sebelumnya

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
spec:
  containers:
    - name: nginx-web-container
      image: nginx
      ports:
        - containerPort: 80
          name: web
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: first-pvc # sesuaikan dengan nama VPC yang sudah dibuat
```

---

## Storage Class

> [!IMPORTANT]
> Jika menggunakan `StorageClass`, maka `PV` akan dibuat dan di-bind otomatis oleh
> Kubernetes

Merupakan suatu Object Kubernetes yang digunakan untuk mendifinisikan tipe storage
yang dapat digunakan secara dinamis oleh app. Secara umum digunakan untuk:

1. Dynamic provisioning
2. Manage Storage Type yang berbeda
   Administrator bisa mendefinisikan beberapa tipe storage berbeda, misalnya:
   - `fast` (SSD)
   - `standard` (HDD)
   - `nfs` (NFS -- Network Storage)
   - `local-only` (storage pada local node)

3. Mengatur Reclaim Policy

4. Specifying Volume Binding Locations
   Mengatur kapan dan dimana volume akan dibuat. Misalnya hingga Pod di scheduled
   pada specific node

Sama seperti PV, Storage Class dapat menggunakan backend berbeda. Mekanisme untuk
mengatur backend berbeda terdapat pada komponen bernama `provisioner`.

### Example Storage Class Local-Path Provisioner

Membuat StorageClass dengan nama `local-storage` dengan backend/provisioner `local-path`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

> [!TIP]
> Atau dapat menggunakan tau install langsung dari template provider/backend.
> Misalnya seperti [Rancher Local Path Provisioner](https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml)

### Example Attach StorageClass to Pod

1. Buat PVC yang mengarah ke storage class yang dibuat sebelumnya terlebih dahulu

   Membuat PVC bernama `pvc-localpath` yang mengarah ke StorageClass `local-path`

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pvc-localpath
   spec:
     storageClassName: local-path # atur disini
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

2. Attach PVC yang sebelumnya dibuat ke Pod

   Membuat Pod bernama `pod-localpath` dengan image `busybox` yang menyimpan text
   `Hello World!` ke `/data/hello`. Kemudian Direktori `/data` di mount ke volume

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-localpath
   spec:
     containers:
       - name: app
         image: busybox
         command:
           ["sh", "-c", "echo 'Hello World!' > /data/hello && sleep 3600"]
         volumeMounts:
           - name: data
             mountPath: /data
     volumes:
       - name: data
         persistentVolumeClaim:
           claimName: pvc-localpath
   ```

---

## Secrets

Secrets dalam Kubernetes digunakan untuk mengelola informasi sensitif seperti
password, API keys, tokens, dll. Hal ini karena informasi sensitif yang disimpan
langsung pada Pod atau volume tidak aman.

Beberapa hal terkait secrets:

- 1MB max size per secret
- Tidak terenkripsi secara default, hanya di encode `base64-encoded`
- Menyediakan beberapa encryption keys

> [!IMPORTANT]
> To encrypt secrets at rest, configure an `EncryptionConfiguration` and start
> `kube-apiserver` with `--encryption-provider-config` params. Recreate secrets
> after enabling encryption.

### Create Secrets via CLI

Membuat secret bernama `mysql-credentials` dengan nilai `password=root`

```sh
kubectl get secrets
kubectl create secret generic mysql-credentials --from-literal=password=root
```

### Create Secrets via YAML

> [!NOTE]
> Nilai dari field `data` perlu di encode terlebih dahulu

Membuat secret bernama `my-secret` dengan nilai `password=root`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  password: cm9vdAo=
```

### Reference Secrets to Pod

Untuk menggunakan atau memanggil Secrets dalam Pod, dapat menggunakan environment
variable atau `env`

Misalnya membuat Pod bernama `secret-pod` dengan image `busybox` yang menampilkan
text `The secret password is $SECRET_PASSWORD`. `$SECRET_PASSWORD` adalah variable
yang atur dengan `env`, dan nilainya sendiri berasal dari secret yang dibuat
sebelumnya, yaitu `my-secret`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: my-secret-container
      image: busybox
      command: ["/bin/sh", "-c", "echo The secret password is $SECRET_PASSWORD"]
      env:
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef: # panggil secret disini
              name: my-secret # nama secret
              key: password # key yang ada dari secret yang dipanggil
```

---
