# Helm Package Manager

- **Date:** July 21th, 2026

## Overview

Eksplorasi dan belajar mengenai Helm di Kubernetes

---

## Helm

why Helm?

Pada environment real-world, aplikasi terdiri dari beberapa komponen yang meliputi
Deployments, Services, ConfigMaps, Sercrets, dan mungkin Persistent Volume (PV)
untuk storage. Setiap komponen ini dikonfigurasi dalam file YAML. Jika aplikasi
terus berkembang, file konfigurasi juga terus bertambah kompleks dan akan memicu
banyak error ketika di kekola secara manual.

**Helm** ada untuk mengatasi ini. Helm adalah sebuah package manager untuk Kubernetes.
Sama seperti `apt` atau `yum` di sistem Linux atau `npm` di Node atau `pip` di Python.
Dengan Helm, memungkinkan untuk me-wrap atau bundle file konfigurasi menjadi satu
reusable package bernama **Chart**.

Fiturnya:

- Easy Sharing
- Simple Customization, dibanding dengan mengubah/megedit YAML, Helm Chart memungkinkan
  untuk melakukan konfigurasi di satu file `values.yaml`
- Version Control & Rollback
- Consistency

### Old Version of Helm

Pada versi awal, Helm menggunakan client-server architecture. Terdapat komponen
bernama Tiller (berjalan sebagai server, di dalam Kubernetes cluster dan berkomunikasi
dengan Kubernetes API) untuk mengelola dan deploy app di Kubernetes cluster, sedangkan
Helm client bisa dijalankan secara local di komputer user. Helm client mendownload
Chart kemudian mengirimkan request ke Tiller.

Tetapi model ini menyebabkan issue di keamanan. Maka Helm v3 memberbaikinya

### Helm v3 and beyond

Pada Helm v3 komponen Tiller dihapus dan di rombak agar lebih stabil, aman, dan
efisien. User hanya berinteraksi dengan Helm CLI, kemudian Helm CLI berkomunikasi
langsung dengan Kubernetes API. Helm v3 juga menggunakan Kubernetes RBAC yang ada
sehingga bisa lebih mudah, simple, aman, dan konsisten.

---

## Chart

Helm Chart merupakan package yang berisi seluruh Kubernetes manifest dan file konfigurasi
yang dibutuhkan untuk deploy aplikasi.

### Chart Structure

```txt


├── Chart.yaml
├── README.md
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── pvc.yaml
│   ├── secrets.yaml
│   └── svc.yaml
└── values.yaml
```

Komponen Chart yaitu:

1. `Chart.yaml`. Metadata utama dari Chart. Berisi:
   - Name of the chart
   - versi
   - appVersion
   - Informasi lain seperti deskripsi, keywords, dan maintainers

2. `values.yaml`. File yang berisi default value untuk konfigurasi

   File ini dapat digunakan untuk mengatur konfigurasi dari aplikasi yang akan
   di deploy di Kubernetes cluster seperti misalnya jumlah replica, port, dll.

3. `templates/`. Direktori yang berisi actual Kubernetes manifest template yang
   mendefinisikan komponen aplikasi yang akan di deploy, misalnya:
   - `deployment.yaml` - defines the Deployment and how Pods are created.
   - `svc.yaml` - defines the Service and how the Pods are exposed.
   - `configmap.yaml` - stores configuration data for the application.
   - `secrets.yaml` - stores sensitive data like passwords or keys.
   - `pvc.yaml` - defines PersistentVolumeClaims for storage.
   - `_helpers.tpl` - contains reusable template functions

> [!IMPORTANT]
> Helm processes these templates at runtime, replacing variables with the values
> provided in `values.yaml` or via the `--set` flag, then generates the final
> Kubernetes manifests to be applied to the cluster.

### How Chart are Used

Helm Chart dapat:

- **Di download** dari public repository (seperti [ArtifactHub](https://artifacthub.io/packages/search?kind=0&sort=relevance&page=1),
  atau Bitnami)
- **Dibuat secara local** dengan `helm create <chart-name>`
- **Shared** dalam suatu organisasi sebagai standarisasi deployment

Ketika Chart siap digunakan, user bisa dengan mudah mendeploy dengan:

```sh
helm install my-release ./mychart
```

> [!NOTE]
> This command deploys all resources defined in the chart into the Kubernetes cluster
> as a release a tracked and versioned instance of the application.

### Templates

The templates are resource manifests that use the Go templating syntax. Variables
defined in the values file, for example, get injected in the template when a release
is created.

Example a set of labels are defined in the Secret metadata using the Chart name,
Release name, etc. The actual values of the passwords are read from the `values.yaml`
file.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: { { template "fullname" . } }
  labels:
    app: { { template "fullname" . } }
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
type: Opaque
data:
  mariadb-root-password:
    { { default "" .Values.mariadbRootPassword | b64enc | quote } }
  mariadb-password: { { default "" .Values.mariadbPassword | b64enc | quote } }
```

---

## Manage Chart

### Install Chart

Untuk install, [RTFM from the official docs](https://helm.sh/docs/intro/install)

verify the installation:

```sh
helm version
```

Jika berhasil di install, maka perintah diatas akan menampilkan output seperti berikut:

```txt
version.BuildInfo{Version:"v4.2.3", GitCommit:"43e8b7feece8beb0fcba47059ec9b522fd929a64",
GitTreeState:"clean", GoVersion:"go1.26.5", KubeClientVersion:"v1.36"}
```

### Working with Chart Repository

Untuk mengelola Chart repository dapat menggunakan perintah `helm repo`. Dengan
perintah ini dapat menambah, list, update, atau hapus repository yang diinginkan.

1. Add Repo

   Untuk menambahkan repository, dapat menggunakan perintah:

   ```sh
   helm repo add bitnami https://charts.bitnami.com/bitnami
   ```

   Hal ini akan menambahkan repository baru bernama `bitnami` dengan URL [https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)

2. List Repo

   Untuk me-list repo, dapat menggunakan perintah:

   ```sh
   helm repo list
   ```

3. Search Repo

   Untuk mencari repo, misalnya repo `redis`, dapat menggunakan perintah:

   ```sh
   helm search redis
   ```

4. Refresh Local Repo

   Untuk me-refresh local repository cache, dapat menggunakan perintah:

   ```sh
   helm repo update
   ```

### Deploying a Chart

1. Deploy

   Untuk deploy Chart, misalnya redis, dapat menggunakan perintah berikut:

   ```sh
   helm install helm-nginx bitnami/nginx
   ```

   - `helm-nginx` merupakan nama deployment
   - `bitnami/nginx` merupakan nama chart dari `bitnami` repository. Pada kasus ini
     chart nya yaitu `nginx`

   > [!NOTE]
   > Ketika command di eksekusi, maka akan:
   >
   > 1. Download Chart dari repo yang telah diatur/dispesifikan
   > 2. Generate release name
   > 3. Membuat semua Kubernetes resource yang dibutuhkan (seperti Pods, Deployments,
   >    Services, dll)
   > 4. Save release info

2. List

   Setiap instalasi di Helm membuat release. Untuk me list release yang ada, dapat
   menggunakan perintah:

   ```sh
   helm list
   ```

   Cek semua Kubernetes resource dengan perintah:

   ```sh
   kubectl get all
   ```

3. Delete

   Untuk men-delete deployment yang ada, dapat menggunakan perintah:

   ```sh
   helm uninstall <nama-release>
   ```

---

## Example Deploying PostgreSQL

Mencoba deploy sebuah PostgreSQL dari repository Bitnami dengan info berikut:

- name deployment: `mypostgres`
- username: `adinusa`
- passwords: `P@ssw0rd`
- PV: `mypostgres-pv`
- PVC: `mypostgres-pvc`

1. Buat PV dan PVC terlebih dahulu

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: mypostgres-pv
   spec:
     storageClassName: ""
     volumeMode: Filesystem
     accessModes:
       - ReadWriteOnce
     capacity:
       storage: 8Gi
     persistentVolumeReclaimPolicy: Retain
     hostPath:
       path: /data/postgres
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mypostgres-pvc
   spec:
     storageClassName: ""
     volumeName: mypostgres-pv
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 8Gi
   ```

   > [!TIP]
   > Karena storage menggunakan HostPath dengan direktori `/data/postgres`,
   > maka **pastikan direktori tersebut ada di setiap node** (atau dapat salah satu
   > node, kemudian menggunakan NodeSelector atau Node Affinity)

2. Deploy PV dan PVC yang sebelumnya didefinisikan

   ```sh
   kubectl apply -f storage.yaml
   ```

3. Deploy PostgreSQL menggunakan Helm dan attach PVC yang sudah dibuat

   ```sh
   helm install mypostgres bitnami/postgresql \
     --set global.postgresql.auth.username=adinusa \
     --set global.postgresql.auth.password=P@ssw0rd \
     --set primary.persistence.existingClaim=mypostgres-pvc
   ```

4. Cek resources dan pastikan Pods `Running`

   ```sh
   kubectl get pods -o wide
   kubectl get deploy
   kubectl get svc
   ```
