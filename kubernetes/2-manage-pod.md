# Kubernetes Manage Pods

- **Date:** July 10th, 2026

## Overview

Mempelajari mengelola pods pada Kubernetes. Bagian dari materi Kubernetes API & Access.

Pods adalah satuan terkecil dalam Kubernetes Cluster. Pods dapat memiliki beberapa
container sekaligus, meskipun umumnya Pods memuat satu container saja.

> 💡 **But why?**
> _The design rationale comes from a concept called tightly coupled processes —
> sometimes two processes need to share the same network namespace and filesystem
> as if they were on the same machine, but you still want them in separate containers
> for isolation and independent builds. So inside Pods, containers communicate with
> localhost via ports (that's why each container within Pod can't bind the same port)
> thanks to `pause containers`_

Atau umumnya satu Pod terdiri dari:

- pause container: untuk mengelola networking
- main/app container: aplikasi berjalan
- sidecar container: helper dari app container. Misalnya updater/db migrator
- metrics agent container: container untuk memonitor resources (seperti RAM/CPU)
  yang digunakan. Misalnya Metrics Server untuk HPA (Horizontal Pod AutoScaling)
- logging agent container: container untuk menyimpan/mengelola log aplikasi. Misalnya
  Prometheus agent

---

## Manage Pods

### 1. List Pods

```sh
kubectl get pods
```

List semua Pods yang tersedia (termasuk pod yang terkait `namespaces`)

```sh
kubectl get pods -A

# atau
kubectl get pods --all-namespaces
```

### 2. Get Detail Pod

Melihat detail pod bernama `httpd`

```sh
kubectl describe pod httpd
```

Melihat label pod bernama `httpd`

```sh
kubectl get pod httpd --show-labels
```

### 3. Create Pod

Untuk membuat resources di Kubernetes, umumnya menggunakan file specs di YAML, baru
kemudian dapat di deploy berdasarkan file YAML tersebut.

Maka untuk membuat pod, perlu mendifinisikan pod specifications dalam YAML. Misal
membuat file `~/simple-pod.yaml`, untuk membuat pod dapat mengikuti format

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd
spec:
  containers:
    - name: httpd
      image: httpd:latest
      ports:
        - containerPort: 80
  restartPolicy: Always
```

Setelah itu, deploy pod berdasarkan file YAML tersebut ke Kubernetes

```sh
kubectl create -f ~/simple-pod.yaml
```

### 4. Delete Pod

Menghapus pod yang sebelumnya dibuat (bernama `httpd`).
Terdapat 2 cara untuk menghapus Pod:

1. Menghapus langsung Pod

   ```sh
   kubectl delete pod httpd
   ```

2. Menghapus melalui template/specs yaml yang sebelumnya digunakan untuk mendeploy

   ```sh
    kubectl delete -f ~/simple-pod.yaml
   ```

Kemudian dapat [cek status pod](#1-list-pods) untuk memastikan bahwa pod sudah
berhasil terhapus.

### 5. Edit Pod

1. Cek pod yang ada/tersedia

   ```sh
   kubectl get pods
   ```

2. Edit Pod langsung (misalnya pod bernama `edit-demo`)

   > [!NOTE]
   > Perintah ini akan membuka/menampilkan file YAML dari Pod yang saat ini berjalan
   > dengan text editor default (umumnya `nano` atau `vim`)

   ```sh
   kubectl edit pod edit-demo
   ```

3. Ubah salah satu bagian metadata atau label

   Misalnya:

   ```yaml
   metadata:
     labels: # bagian ini yang ditambahkan
       app: web
       env: dev
   ```

4. Kemudian coba ubah lagi bagian metadata `name`

   Misalnya:

   ```yaml
   metadata:
     name: edit-test-renamed
   ```

   Pada tahap ini, akan menampilkan error berikut:

   ```txt
   A copy of your changes has been stored to "/tmp/manifest-pod.yaml"
    error: At least one of apiVersion, kind and name was changed
   ```

   > [!TIP]
   > Hal ini terjadi karena beberapa komponen tidak bisa diubah langsung karena
   > hal ini akan mengubah identitas Pod seutuhnya. Maka dari itu, cara terbaik
   > untuk mengedit metadata name/kind atau apiVersion yaitu dengan menghapus Pod
   > saat ini. Mengedit file YAML yang berkaitan, kemudian menjalankan nya kembali

### 6. Check Pod Logs

Check log pada Pod `httpd`

```sh
kubectl logs httpd
```

### 7. Connect and Interact with Pod

Connect ke Pod `httpd` dan akses bash/shell

```sh
kubectl exec -it httpd -- /bin/bash
```
