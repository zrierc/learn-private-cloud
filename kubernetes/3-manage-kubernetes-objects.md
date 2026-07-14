# Kubernetes Objects

- **Date:** July 13th, 2026

## Overview

Eksplorasi dan belajar mengenai Kubernetes Object:

1. Namespace
2. RBAC
3. DaemonSet
4. AutoScaling
5. Jobs
6. StatefulSet

Topologi lihat di [catatan sebelumnya](./1-setup-intallation.md).

---

## Namespace

Namespace adalah logical group yang dapat digunakan untuk mengelola atau mengelompokan
resources dalam cluster. Hal ini bisa digunakan untuk melakukan isolasi terhadap
resources seperti Pod, Services, dan ConfigMap.

> [!TIP]
> Bisa dimanfaatkan untuk pemisah environment seperti development, staging, dan production

Secara default terdapat beberapa Namespace di dalam suatu cluster, diantaranya:

- `default`, digunakan ketika ada resources yang tidak menggunakan Namespace
- `kube-node-lease`, berisi object sementara (Lease Objects) untuk setiap node.
  Umumnya digunakan control plane untuk mendeteksi kegagalan node via Kubelet heartbeats
- `kube-public`, Readable oleh semua user meski tanpa autentikasi, jarang digunakan
- `kube-system`, berisi resources yang dibuat dan dikelola langsung oleh Kubernetes
  (misalnya kube-dns, CNI pods, etc)

### 1. View All Namespace

```sh
kubectl get namespaces
```

atau bisa disingkat

```sh
kubectl get ns
```

### 2. Create new Namespace

Membuat Namespace bernama `backendTeam`

```sh
kubectl create ns backendTeam
```

### 3. Describe (show details) Namespace

Lihat detail (meliputi label, annotations, quota, limit) dari Namespace `backendTeam`

```sh
kubectl describe ns backendTeam
```

### 4. View Namespace Configuration

Lihat konfigurasi Namespace `backendTeam` dalam format YAML

```sh
kubectl get ns backendTeam -o yaml
```

> [!TIP]
> Untuk melihat dalam format JSON, ubah `-o yaml` menjadi `-o json`

### 5. Delete Namespace

> [!CAUTION]
> Menghapus Namespace akan menghapus seluruh resources yang ada di dalamnya

Menghapus namespace `backendTeam`

```sh
kubectl delete ns backendteam
```

### 6. Using Namespaces in YAML

Namespace bisa digunakan ketika membuat resources, misalnya menggunakan Namespace
`backendTeam` untuk membuat Pod dengan mendefinisikannya pada properti `metadata`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: backendteam # Sesuaikan dengan Namespace yang diinginkan
```

---

## RBAC

Role-Based Access Control (RBAC) dalam Kubernetes digunakan untuk membatasi siapa
yang boleh akses dan melakukan operasi (actions) kepada sebuah resources dalam cluster.

RBAC menentukan:

- Who (subject): user, group, service account
- Can do what: actions seperti `get`, `list`, `describe`, `create`, `update`, `delete`
- On which resources: `Pods`, `Deployments`, `Nodes`, `Services`, etc
- In what scope: spesifik Namespace atau seluruh cluster

RBAC memiliki 4 main API (`rbac.authorization.k8s.io`) resources

| Component            | Scope        | Tujuan                                                                                                     |
| -------------------- | ------------ | ---------------------------------------------------------------------------------------------------------- |
| `Role`               | Namespaced   | Mendifinisikan set of permissions dalam sebuah Namespace                                                   |
| `ClusterRole`        | Cluster-wide | Sama seperti `Role`, tetapi scope nya cluster                                                              |
| `RoleBinding`        | Namespaced   | Mengasosasikan `Role` atau `ClusterRole` dengan subject (user, group, service account) ke sebuah Namespace |
| `ClusterRoleBinding` | Namespaced   | Sama seperti `RoleBinding` tetapi scope nya cluster                                                        |

Beberapa contohnya:

> [!TIP]
> Contoh dalam bentuk YAML. Untuk apply simpan dalam file kemudian apply ke Kubernetes.
> `kubectl apply -f rbac.yaml`

### 1a. Role Example

Membuat Role bername `pod-reader` dengan akses Read Pods dalam default Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

### 1b. RoleBinding Example

Assign Role `pod-reader` ke user bernama `rei`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: User
    name: rei
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2a. ClusterRole Example

Membuat ClusterRole bernama `cluster-admin-role` dan memberikan full access Pods

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["*"]
```

### 2b. ClusterRoleBinding Example

Assign ClusterRole `cluster-admin-role` ke user bernama `shinji`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: shinji
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin-role
  apiGroup: rbac.authorization.k8s.io
```

---

## DaemonSet

DaemonSet merupakan Kubernetes Object memastikan agar Pods berjalan di setiap node
(atau node yang dipilih) dalam suatu cluster. Umumnya digunakan untuk task tertentu
seperti log collection, monitoring agents, atau komponent networking (seperti CNI)
yang harus berjalan di setiap node. Ketika menggunakan DaemonSet:

- Node baru ditambahkan: DaemonSet akan otomatis schedules Pod di node tersebut
- Node di hapus: DaemonSet akan otomatis clean up Pod
- DaemonSet di hapus: Semua Pods yang dibuat oleh DaemonSet tersebut akan ikut
  hapus juga

### 1. Check DaemonSet in Cluster

```sh
kubectl get daemonsets
```

atau bisa disingkat

```sh
kubectl get ds
```

### 2. Example DaemonSet

Membuat DaemonSet untuk logging bername `log-agent` dengan base image `fluentd`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
        - name: log-agent
          image: fluent/fluentd
```

> [!TIP]
> Contoh dalam bentuk YAML. Untuk apply simpan dalam file kemudian apply ke Kubernetes.
> `kubectl apply -f daemonsets.yaml`

---

## AutoScaling

AutoScaling digunakan untuk melakukan horizontal scaling (scale-in dan scale-out)
Pods secara otomatis berdasarkan metrics tertentu dalam sebuah ReplicationController,
ReplicaSet, atau Deployment

- Default target: 50% CPU utilization

Cara kerja:

- Metrics Collection
  - Kubelet check CPU usage setiap `30s`
  - Metrics Server updates setiap `1s`
  - HPA queries the Metrics Server setiap `30s`

- Scaling Delay
  - Setelah add/remove sebuah Pod, HPA menunggu `180s` sebelum scaling decision berikutnya

- Custom Metrics
  - HPA juga bisa menggunakan custom / external metrics melalui REST API

> [!NOTE]
> HPA **TIDAK** me-collect metrics. HPA hanya requests aggregated data, kemudian
> adjust jumlah replica agar match dengan target

### Cluster Autoscaler (CA)

> [!NOTE]
> AutoScale untuk scale Pod. Cluster Autoscaler (CA) untuk auto-scale node.

- Tujuan: Adjust jumlah nodes dalam cluster
- When it scale up:
  - Pod tidak dapat di scheduled karena resources yang minim (misal RAM penuh,
    penggunaan CPU tinggi)
- When it scale down:
  - Node memiliki utilization yang rendah setidaknya `10m`
- Checks:
  - Scale-up/down check muncul setiap `10s`
  - Penentuan scale-down muncul setiap `10m`
- Failures & Retries
  - Jika scale-down gagal, CA retries dalam `3m`
  - Node yang gagal menjadi eligible untuk dihapus setelah `5m`
- Cloud Dependence
  - Waktu yang dibutuhkan untuk menambah node tergantung berdasarkan kecepatan
    cloud provider saat provisioning instance/VM/server
- Best Practice
  - Gunakan `cluster-autoscaler` commands untuk menambahkan/menghapus node
    (jangan menggunakan manual node management)

### Metrics Server

Digunakan untuk mengetahui resources setiap Pod dan Node yang digunakan sehingga
bisa memanfaatkan dan memaksimalkan fitur HPA (Horizontal Pod Autoscaling)

1. Download Metrics Server

   ```sh
   wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

2. Edit file `components.yaml` yang mendefinisikan Metrics Server

   Tambahkan parameter `--kubelet-insecure-tls` agar metrics server bisa berkomunikasi
   dengan Kubelet menggunakan koneksi non-TLS. Ubah port `10250` menjadi `4443` untuk
   menghindari port conflict yang digunakan oleh komponen Kubernetes. Ubah juga
   port `https` menjadi `4443`.

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     labels:
       k8s-app: metrics-server
     name: metrics-server
     namespace: kube-system
   spec:
     # ...
     specs:
       containers:
         - args:
             - --cert-dir=/tmp
             - --secure-port=4443
             - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
             - --kubelet-use-node-status-port
             - --metric-resolution=15s
             - --kubelet-insecure-tls
           image: registry.k8s.io/metrics-server/metrics-server:v0.6.4
           # ...
           livenessProbe:
             failureThreshold: 3
             httpGet:
               path: /livez
               port: 4443 # here
               scheme: HTTPS
             periodSeconds: 10
             name: metrics-server
             ports:
               - containerPort: 4443 # here
                 name: https
                 protocol: TCP
             readinessProbe:
               failureThreshold: 3
               httpGet:
                 path: /readyz
                 port: 4443 # here
                 # ...
               # ...
             # ...
   ```

3. Deploy metrics-server dalam cluster

   ```sh
   kubectl apply -f components.yaml
   ```

### HPA Simulation

Lakukan simulasi auto-scale dengan membuat Deployment sederhana dengan container
Nginx

1. Buat Deployment file

   ```sh
   vim autoscaling.yaml
   ```

   Buat Deployment (bernama `lab52-deployment`)

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: lab52-deployment
   spec:
     replicas: 1
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
             resources:
               requests:
                 cpu: "100m"
               limits:
                 cpu: "200m"
             ports:
               - containerPort: 80
   ```

2. Apply Deployment

   ```sh
   kubectl apply -f autoscaling.yaml
   ```

3. Create service deployment

   Digunakan untuk me-expose container yang ada deployment sebelumnya, agar bisa
   diakses dari host dengan format `NodeIP:NodePort`.

   ```sh
   kubectl expose deployment lab52-deployment --type=NodePort --name=nginx-service --port=80
   ```

4. Create Auto Scale (HPA) Configuration

   ```sh
   kubectl autoscale deployment lab52-deployment --cpu-percent=10 --min=1 --max=10
   ```

   Akan melakukan auto-scale pada deployment bernama `lab52-deployment` dengan
   minimum 1 Pod dan maximum 10 Pod. Threshold yang digunakan yaitu CPU 10%. Sehingga
   HPA akan me-maintain avg CPU usage sebesar 10%. Jika lebih dari 10%, akan melakukan
   scale-out dengan menambah replica. Sebaliknya, jika kurang dari 10%, akan
   melakukan scale-in dengan mengurangi replica.

5. Verify HPA

   ```sh
   kubectl get hpa
   ```

6. Check Service untuk mengetahui Port yang dapat diakses

   ```sh
   kubectl get svc -o wide
   ```

   Contoh output:

   ```txt
   NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE    SELECTOR
   kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        5d6h   <none>
   nginx-service   NodePort    10.106.115.113   <none>        80:31132/TCP   3m6s   app=nginx
   ```

   service `nginx-service` yang sebelumnya dibuat dapat diakses di Port `31132`

7. Check Node yang menjalankan Pod nginx tersebut

   ```sh
   kubectl get pods -o wide
   ```

   Contoh output:

   ```txt
   NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE              NOMINATED NODE   READINESS GATES
   lab52-deployment-7c874f49c8-488mp   1/1     Running   0          2m43s   10.244.1.33   rei-pod-worker1   <none>           <none>
   ```

   Maka Pod tersebut dijalankan di node `rei-pod-worker1`. Maka untuk aksesnya
   yaitu dengan `[IP_rei-pod-worker1]:[Port-Service]`.

8. Akses web yang di serve oleh Nginx

   Port yang dapat diakses yaitu `31132`. Misal node `rei-pod-worker1` memiliki
   IP `10.1.1.20`. Maka untuk aksesnya:

   ```sh
   curl http://10.1.1.20:31132/
   ```

   Jika berhasil akan me-return HTML default Nginx. Tahap ini dapat dilakukan
   berkali-kali atau dapat menggunakan stress-test tools seperti `ab` (Apache
   Benchmark) agar CPU usage setiap Node yang me-serve service tersebut meningkat.

9. Lihat dan perhatikan HPA dan Deployment

   Lihat HPA secara live untuk melihat CPU usage & jumlah replica yang diterapkan

   ```sh
   kubectl get hpa -w
   ```

   Untuk lihat Deployment secara live untuk lihat jumlah replica/pod yang saat  
   ini berjalan

   ```sh
   kubectl get deployment -w
   ```

---

## Jobs

Jobs adalah object Kubernetes dalam kategori batch API group yang dapat digunakan
untuk menjalankan task spesifik yang sekali jalan (memiliki lifetime) karena Jobs
hanya akan dijalankan satu waktu saja. Hal ini berbanding terbalik dengan Deployment
yang mempertahankan Pods agar berjalan secara continue.

Biasanya digunakan untuk batch atau one-time tasks, seperti calculation atau data
processing. Jobs dapat di kontrol berapa banyak Pod yang dapat digunakan dengan
field `Parallelism` dan berapa banyak completions yang dibutuhkan dengan field
`completions`.

### 1. List Jobs

Lihat informasi terkait Jobs yang dibuat, beserta jumlah completions dan status
saat ini.

```sh
kubectl get jobs
```

### 2. Example Jobs

Membuat Job sederhana dengan nama `first-job`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: first-job
spec:
  template:
    metadata:
      name: first-pod
    spec:
      containers:
        - name: first-job-container
          image: busybox:latest
          command: ["echo", "Hello, Rei!"]
      restartPolicy: OnFailure
  backoffLimit: 4
```

- Job tersebut menggunakan image `busybox`
- Menampilkan teks `Hello, Rei!`
- `backoffLimit` digunakan untuk mendefinisikan jumlah retries maximum 4 kali ketika
  Job gagal.

> [!TIP]
> Contoh dalam bentuk YAML. Untuk apply simpan dalam file kemudian apply ke Kubernetes.
> `kubectl apply -f jobs.yaml`

Untuk melihat hasil Jobs:

1. Lihat Pods yang menjalankan Job

   ```sh
   kubectl get pods
   ```

2. Lihat Job logs

   ```sh
   kubectl logs <pod-name>
   ```

   Jika berhasil maka Log akan menampilkan `Hello, Rei!`

---

## StatefulSet
