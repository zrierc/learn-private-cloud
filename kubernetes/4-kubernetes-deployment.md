# Kubernetes Deployment

- **Date:** July 14th, 2026

## Table of Contents

<!--toc:start-->

- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [State](#state)
  - [Desired State](#desired-state)
  - [Actual State](#actual-state)
- [Top Level Configuration](#top-level-configuration)
- [Metadata Configuration](#metadata-configuration)
- [Spec Configuration](#spec-configuration)
- [Pod Template Configuration](#pod-template-configuration)
- [Status Configuration](#status-configuration)
- [Scaling and Rolling Updates](#scaling-and-rolling-updates)
  - [Scaling](#scaling)
  - [Rolling Updates](#rolling-updates)
- [Rollback](#rollback)
  - [Create Deployment with change record](#create-deployment-with-change-record)
  - [Lihat History dari Deployment](#lihat-history-dari-deployment)
  - [Rollback ke satu versi sebelumnya](#rollback-ke-satu-versi-sebelumnya)
  - [Rollback ke spesifik versi](#rollback-ke-spesifik-versi)
  - [Other Rollout Commands](#other-rollout-commands)
- [Labels](#labels)
  - [Deep Down into Selector](#deep-down-into-selector)
  - [List Pods with Labels](#list-pods-with-labels)
  - [View Pods by Label](#view-pods-by-label)
  - [Add Labels to Existing Object](#add-labels-to-existing-object)
  <!--toc:end-->

---

## Overview

> [!NOTE]
> Topologi lihat di [catatan sebelumnya](./1-setup-intallation.md).

Eksplorasi dan belajar mengenai Kubernetes Deployment dan format YAML yang digunakan
pada Kubernetes Object.

Deployment sendiri merupakan Kubernetes object yang digunakan untuk manage dan update
aplikasi. Dalam Deployment, di definisikan container image yang akan digunakan,
berapa banyak Pods seharusnya berjalan, behaviour ketika ada update / perubahan
pada aplikasi, dsbnya.

Deployment Object terdiri dari: Deployment -> ReplicaSet -> []Pods

Deployment dapat dibuat, secara sederhana, dengan `kubectl` seperti

```sh
kubectl create deployment dev-web --image=nginx:latest
```

Tetapi umumnya (dan juga best practice), Deployment dibuat dalam sebuah file YAML
sehingga bisa persistent dan mudah di maintain. Kemudian baru di deploy/apply

```sh
kubectl apply -f /path/to/deployment.yaml
```

Detail dokumentasi YAML reference untuk Deployment terdapat di:
[https://kubernetes.io/docs/reference/kubernetes-api/apps/deployment-v1/](https://kubernetes.io/docs/reference/kubernetes-api/apps/deployment-v1/)

---

## State

### Desired State

Desired state adalah konfigurasi yang di definisikan oleh user/dev.

> [!TIP]
> Untuk mengetahui field yang wajib/diperlukan dengan yang opsional, di dalam dokumentasi
> (atau melalui `kubectl explain`), field yang wajib ditandai dengan `required`.
> Jika secara explisit tidak disebutkan apapun, maka field tersebut bersifat opsional.

### Actual State

Actual state adalah konfigurasi yang telah di deploy/apply dan berjalan di Kubernetes
cluster berdasarkan desired state yang telah ditentukan sebelumnya.

Terdapat perbedaan, dimana Actual state akan memberikan lebih banyak informasi
berupa property/field tamabahan pada konfigurasi YAML.

Misalnya ketika sudah di deploy/apply, konfigurasi Deployment akan ada field yang
ditambah oleh Kubernetes sendiri untuk memberikan informasi / state saat berjalan.
Beberapa diantaranya yaitu `uid`, `creationTimestamp`, dsbnya tidak di definisikan
ketika dibuat. Tetapi ketika di cek akan muncul, hal ini karena Kubernetes yang
menambahkannya langsung.

> [!TIP]
> Untuk membedakan antara Actual State vs Desired State, di dalam dokumentasi
> Kubernetes (atau melalui `kubectl explain`), Actual state ditandai dengan:
> `"Populated by the system"`, `"Read-only"`. Artinya field tersebut diatur oleh
> Kubernetes, jika user/dev yang mengaturnya, maka akan diabaikan dan nilainya
> ditimpa oleh Kubernetes

---

## Top Level Configuration

Setiap Kubernetes Object, apapun tipenya, setidaknya memiliki 4 field/property ini
ketika mendefinisikan konfigurasinya:

```yaml
apiVersion:
kind:
metadata:
spec:
```

- `apiVersion`: API Group dan version yang handle object yang akan dibuat
- `kind`: Tipe dari object yang akan dibuat
- `metadata`: identitas/informasi (seperti nama, namespace, labels, annotation, etc)
- `spec`: _desired state_, aktual konfigurasi dari Object yang akan dibuat

> [!NOTE]
> Deployment sendiri merupakan bagian dari `apps` API Group. Sehingga untuk
> mendefinisikan Deployment dapat menggunakan `apiVersion: apps/v1`

## Metadata Configuration

```yaml
# ...
Kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: 2022-09-12T04:07:07Z
  generation: 1
  labels:
    app: dev-web
  name: dev-web
  namespace: default
  resourceVersion: "774003"
  selfLink: /apis/apps/v1/namespaces/default/deployments/dev-web
  uid: d52d3a63-e656-11e7-9319-42010a800003
```

Field:

- `annotations`: Extra notes or data used by tools or for tracking purposes;
  not used to select objects.
- `creationTimestamp`: The exact date and time when this object was first created.
  **Populated by system. Read-only.**
- `generation`: A counter that increases each time the object’s configuration
  is updated. **Populated by system. Read-only.**
- `labels`: Key-value tags that help you group, filter, or find objects easily.
- `name`: The object’s unique name within its namespace — it’s **required**
  for all resources.
- `resourceVersion`: A number used internally by Kubernetes to track updates to
  the object. **Populated by system. Read-only.**
- `selfLink`: The API path that points directly to this object (used more in
  older Kubernetes versions). **Populated by system. Read-only.**
- `uid`: A permanent unique ID assigned to the object that never changes, even
  if the name is reused later. **Populated by system. Read-only.**

## Spec Configuration

```yaml
# ...
Kind: Deployment
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: dev-web
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template: <PodTemplateSpec>
```

Field:

- `spec`: Defines the setup and desired state for this Deployment.
- `progressDeadlineSeconds`: How long Kubernetes waits (in seconds) before marking
  the Deployment as failed if progress stalls (for example, due to a bad image
  or resource issue).
- `replicas`: The number of Pods that should always be running. Increasing this
  number creates more Pods automatically.
- `revisionHistoryLimit`: How many old `ReplicaSets` Kubernetes should keep in case
  you need to roll back to a previous version.
- `selector`: Tells the Deployment which Pods it manages, based on their labels.
  Don't manually create Pods that share these same labels to avoid conflicts.
- `matchLabels`: The specific label values Kubernetes looks for when linking Pods
  to this Deployment.
- `strategy`: Explains how updates happen when you change the Deployment.
- `maxSurge`: The maximum number of extra Pods that can be created temporarily
  during an update.
- `maxUnavailable`: The maximum number of Pods that can be down during an update.
- `type`: The method of updating Pods — usually `RollingUpdate` (update gradually)
  or Recreate (stop old Pods first, then start new ones).

## Pod Template Configuration

```yaml
# ...
Kind: Deployment
spec:
  template:
    creationTimestamp: null
    metadata:
      labels:
        app: dev-web
    spec:
      containers:
        - image: nginx:1.13.7-alpine
          imagePullPolicy: IfNotPresent
          name: dev-web
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

Field:

- `template`: Defines what each Pod created by the ReplicaSet should look like.
- `containers`: Lists all containers that will run inside each Pod.
- `image`: The container image that will be used (Kubernetes pulls it if it’s not
  already on the node).
- `imagePullPolicy`: Determines when Kubernetes should pull a new image (e.g.,
  `IfNotPresent` means it only pulls if not already cached).
- `name`: The base name of the Pod; Kubernetes adds a unique suffix to each one.
- `resources`: Optional CPU and memory requests or limits (empty by default).
- `terminationMessagePath`: The file where the container writes its shutdown or
  error messages.
- `terminationMessagePolicy`: Defines how Kubernetes collects those termination messages.
- `dnsPolicy`: Configures how DNS works for the Pod (ClusterFirst uses cluster DNS).
- `restartPolicy`: Tells Kubernetes when to restart containers (Deployments
  always use Always).
- `schedulerName`: Specifies which scheduler decides where to place the Pod.
- `securityContext`: Defines security options, like user IDs or SELinux/AppArmor
  settings.
- `terminationGracePeriodSeconds`: How long Kubernetes waits after sending a stop
  signal before forcefully ending the container.stopping the container.

## Status Configuration

> [!NOTE]
> Bagian `status` memperlihatkan state secara real-time dari suatu Deployment.
> Bagian ini otomatis di update oleh Kubernetes.

```yaml
kind: Deployment
status:
  availableReplicas: 2
  conditions:
    - lastTransitionTime: 2022-09-12T04:07:07Z
      lastUpdateTime: 2022-09-12T04:07:07Z
      message: Deployment has minimum availability.
      reason: MinimumReplicasAvailable
      status: "True"
      type: Available
  observedGeneration: 2
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2
```

Field:

- `availableReplicas`: Shows how many Pods are currently running and available,
  as managed by the `ReplicaSet`. You can compare this with `readyReplicas` to confirm
  if all Pods have started successfully and are running without errors.

- `observedGeneration`: Indicates how many times the Deployment has been updated.
  This helps track changes during updates or rollbacks.

- `replicas`, `readyReplicas`, `updatedReplicas`: Show how many total Pods exist,
  how many are ready, and how many have been updated to the newest version.

- `conditions`: Provides detailed messages and timestamps about the current state,
  such as whether the Deployment is available or progressing.

---

## Scaling and Rolling Updates

### Scaling

Scaling digunakan untuk menentukan/adjust jumlah Pods yang running dalam suatu
Deployment. Dapat melakukan scale-in atau scale-out.

Untuk melakukan scaling, misalnya melakukan scaling pada Deployment bernama `dev-web`
dengan `4` replica Pods:

```sh
kubectl scale deploy/dev-web --replicas=4
```

> [!NOTE]
> Scaling ke `0` menghapus semua Pods tetapi tetap mempertahankan `ReplicaSet`
> dan `Deployment` (mirip seperti deletion process)

### Rolling Updates

Rolling Updates digunakan untuk mengatur mekanisme dan behaviour dalam mengganti
Pods secara bertahap, dibanding dengan memberhentikan semuanya dalam satu waktu
kemudian menyalakannya lagi.

Misalnya, me-trigger rolling update dengan mengganti image container pada Deployment.

- Edit langsung deployment bernama `nginx`

  ```sh
  kubectl edit deployment nginx
  ```

- Ubah image atau versi dari image

  ```yaml
  containers:
    - image: nginx:1.8 # older version
      imagePullPolicy: IfNotPresent
      name: dev-web
  ```

  Setelah di save, maka akan me-trigger rolling update. Deployment age akan di reset,
  Pod akan menggunakan image yang terbaru (versi `1.8`) secara bertahap

## Rollback

Kubernetes menyimpan history dari setiap perubahan pada Deployment, hal inilah
yang memungkinkan Kubernetes untuk melakukan rollback suatu Deployment ke versi
sebelumnya.

Hal ini dapat digunakan misalnya ketika mencoba fitur baru pada aplikasi, jika
fiturnya tidak berjalan dengan baik atau membuat kendala pada seluruh aplikasi,
maka aplikasi dapat di rollback ke versi sebelumnya yang belum memiliki fitur
tersebut.

### Create Deployment with change record

Untuk memberitahu Kubernetes agar setiap pembuatan deployment disimpan sehingga
dapat di track atau di audit dapat menggunakan `--record`, misalnya:

```sh
kubectl create deploy ghost --image=ghost --record
```

Maka Kubernetes akan membuat Deployment dengan annotations:

```yaml
kind: Deployment
metadata:
  annotation:
    deployment.kubernetes.io/revision: "1"
    kubernetes.io/change-cause: kubectl create deploy ghost --image=ghost --record
```

### Lihat History dari Deployment

Melihat version history Deployment bernama `ghost`

```sh
kubectl rollout history deployment/ghost
```

### Rollback ke satu versi sebelumnya

Melakukan rollback pada Deployment bernama `ghost`

```sh
kubectl rollout undo deployment/ghost
```

### Rollback ke spesifik versi

Melakukan rollback ke versi `2` pada Deployment bernama `ghost`

```sh
kubectl rollout undo deployment/ghost --to-revision=2
```

### Other Rollout Commands

melakukan rollout pada deployment bernama `ghost`

1. Pause Deployment

   ```sh
   kubectl rollout pause deployment/ghost
   ```

2. Resume Deployment

   ```sh
   kubectl rollout resume deployment/ghost
   ```

---

## Labels

> [!IMPORTANT]
> Label bukanlah API object

Label adalah key-value pairs yang dapat ditambahkan di object metadata. Label
digunakan untuk organize, grouping, atau memilih/select resource seperti melakukan
filtering atau mencari resources.

Pada Deployment, labels digunakan oleh Kubernetes untuk mengelola Pod, field
`selector` memberitahu kepada Kubernetes agar setiap Pod yang memiliki informasi
labels tertentu, maka akan dikelola dan mengikuti rules Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-logging-thing # Nama Deployment
spec:
  selector:
    matchLabels:
      app: log-agent # Deployment akan mengelola pods yang memiliki label ini
  template:
    metadata:
      labels:
        app: log-agent # Memberi stamps pods yang akan dibuat dengan label ini
```

> [!TIP]
> Label pada `selector` dan `template` **harus sama**. Jika tidak Kubernetes akan
> menampilkan error ketika men-deploy/apply.

### Deep Down into Selector

Selector memiliki banyak jenis, misalnya seperti `matchLabels`.

- `AND` selector

  `matchLabels` mendukung multiple label, misalnya:

  ```yaml
  selector:
    matchLabels:
      app: backend
      environment: prod
      version: v2
  ```

  Artinya, selector akan memilih label `app=backend` AND `environment=prod` AND `version=v2`

- `OR` selector

  Jika membutuhkan logika OR, maka dapat menggunakan `matchExpressions` dengan
  `In` operator

  ```yaml
  selector:
    matchExpressions:
      - key: app
        operator: In # OR within values list
        values:
          - backend
          - api
  ```

  Artinya selector akan memilih label `app=backend` OR `app=api`

- `NOT` selector

  Jika membutuhkan logika OR, maka dapat menggunakan `matchExpressions` dengan
  `NotIn` operator

  ```yaml
  selector:
    matchExpressions:
      - key: environment
        operator: NotIn # must NOT be any of these
        values:
          - dev
          - test
  ```

  Artinya selector akan memilih label yang **BUKAN** `environment=dev` atau `environment=test`

- Key Selector

  Selector juga dapat memilih suatu key tanpa peduli nilainya dengan cara menggunakan
  `matchExpressions` dengan operator `Exists`

  ```yaml
  selector:
    matchExpressions:
      - key: version
        operator: Exists # key must exist, value doesn't matter
  ```

  Artinya selector akan memilih Pods dengan label yang bernama `version` tanpa
  peduli nilai dari `version` itu sendiri

- Exclude Key Selector

  Selain memilih suatu key saja, Selector juga dapat me-exclude key tertentu dengan
  cara menggunakan `matchExpressions` dengan operator `DoesNotExists`

  ```yaml
  selector:
    matchExpressions:
      - key: deprecated
        operator: DoesNotExist # key must not exist at all
  ```

  Artinya selector akan memilih label yang **tidak memiliki** key bernama `deprecated`
  apapun nilainya.

- Complex Selector

  Selector dapat dikombinasikan untuk memilih label. Misalnya ingin memilih labels
  berikut: `team=platform AND (environment=prod OR environment=staging) AND
tidak memiliki deprecated keys`

  Maka rules diatas dapat ekspresikan dengan:

  ```yaml
  selector:
  matchLabels:
    team: platform # AND
  matchExpressions:
    - key: environment
      operator: In
      values: [prod, staging] # AND
    - key: deprecated
      operator: DoesNotExist # AND
  ```

### List Pods with Labels

```sh
kubectl get pods --show-labels
```

### View Pods by Label

Memfilter Pods dengan label `run=ghost`

```sh
kubectl get pods -l run=ghost
```

Lihat label sebagai kolom

```sh
kubectl get pods -L run   # shows labels as columns
```

### Add Labels to Existing Object

Menambahkan label `foo=bar` pada pod bernama `ghost-3378155678-eq5i6`

```sh
kubectl label pods ghost-3378155678-eq5i6 foo=bar
```
