# Kubernetes Scheduler

- **Date:** July 21th, 2026

## Table of Contents

<!--toc:start-->

- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Scheduler](#scheduler)
- [Priority in Kubernetes](#priority-in-kubernetes)
  - [PriorityClass](#priorityclass)
  - [Preemption Mechanism](#preemption-mechanism)
  - [Node Labeling](#node-labeling)
- [Node Affinity](#node-affinity)
  - [Rules](#rules)
  - [Example Node Affinity](#example-node-affinity)
- [Pod Affinity](#pod-affinity)
  - [Example Pod Affinity](#example-pod-affinity)
- [Pod Anti-Affinity](#pod-anti-affinity)
  - [Example Pod Anti-Affinity](#example-pod-anti-affinity)
- [Taints](#taints)
  - [Example Taints](#example-taints)
- [Toleration](#toleration) - [Example Toleration](#example-toleration)
  <!--toc:end-->

## Overview

Eksplorasi dan belajar mengenai Scheduler di Kubernetes

> [!IMPORTANT]
> Lihat topology di [catatan sebelumnya](./1-setup-intallation.md).

---

## Scheduler

> [!NOTE]
> Scheduler di Kubernetes bukan mengacu pada time-based scheduler seperti cronjob.

Scheduler atau lebih spesifik `kube-scheduler` merupakan core component Kubernetes
yang bertujuan untuk mengelola dan menjadwalkan (_scheduling_) Pod dan Node. Seperti
Node mana yang harus menjalankan setiap Pod yang ada

> [!TIP]
> Jika sudah mengerti, baca [topik menarik terkait scheduler](./findings/scheduler-rules-priority-and-conflict-each-other.md)

---

## Priority in Kubernetes

Umumnya, Scheduler menentukan dimana Pod akan dijalan, berdasarkan beberapa faktor:

- Resources Availability (CPU, RAM)
- Node Affinity & Anti-Affinity
- Tolerations & taints
- Load distribution accross nodes
- PriorityClass (jika Resources sangat terbatas)

Priority digunakan jika resources sangat terbatas, sehingga Scheduler perlu mengetahui
Pod mana yang lebih penting untuk dipertahakan dan dijalankan lebih awal.

### PriorityClass

`PriorityClass` merupakan sebuah Kubernetes Object untuk mendefinisikan level
priority pada Pods. Prioritas dengan **semakin tinggi nilai prioritasnya, semakin
tinggi/penting juga Pod**

Example:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Pods with high priority may displace other Pods"

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-high
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```

### Preemption Mechanism

Preemption merupakan mekanisme scheduler untuk menghapus low-priority Pods untuk
memberikan resources/ruang untuk high-priority Pods

Alurnya:

1. Pod dengan high-priority tidak bisa diakomodasi karena resource full
2. Scheduler mencari Pods dengan low priority yang bisa dilepas
3. Pod dilepas/dihapus sehingga resource dapat digunakan oleh high-priority Pod
4. Kubernetes reschedule high-priority Pod ke Node yang berkaitan

Untuk mengatasi/menjaga agar suatu pod tidak bisa otomatis dilepas oleh scheduler,
meskipun priority nya lebih rendah, maka dapat menggunakan field

```yaml
preemptionPolicy: Never
```

### Node Labeling

Umumnya label digunakan untuk filtering oleh Scheduler. Hal ini termasuk Node.
Label adalah `key=value` pairs. Untuk melabeli Node dapat dengan perintah:

```sh
kubectl label nodes <nama-node> disk=ssd
```

Perintah diatas digunakan untuk melabeli Node dengan key `disk` yang memiliki value
`ssd`.

---

## Node Affinity

Node Affinity merupakan versi terbaru dan advance filter, dari `NodeSelector`.
Node Affinity digunakan oleh **Pod spec** (Pod -> Node). Sehingga Pod dapat menentukan
Node mana yang dipilihnya.

Ilustration

```txt
Pod has nodeAffinity → "I only want nodes with disktype=ssd"
Node has labels      → "I am disktype=ssd"

Scheduler reads both and matches them.
The pod is the one expressing the requirement.
The node is passive — it just has labels.
```

### Rules

- Node Affinity dapat menggunakan operator `In`, `NotIn`, `Exists`, `DoesNotExist`.
  Hal ini memungkinkan melakukan advance filter dengan berbagai kondisi seperti AND,
  OR, NOT, dan kombinasinya.

- Field `requiredDuringSchedulingIgnoredDuringExecution`
  Digunakan untuk mendefinisikan agar Pod tidak akan di-schedule pada sebuah Node
  kecuali operator nya `true`. Disebut **hard rule**.

- Field `preferredDuringSchedulingIgnoredDuringExecution`
  Digunakan untuk mendefinisikan agar Pod di-schedule pada Node dengan mengikuti
  operator nya, tetapi bila tidak terpehuni (operator nya `false`) Pod tetap akan
  dijalankan. Disebut **soft rule / soft setting**.

### Example Node Affinity

Contoh Node Affinity dengan rule:

The Pod must run on one of the specified data center nodes (tx-aus or tx-dal),
and if multiple options exist, Kubernetes will prefer nodes with faster disks.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/colo-tx-name
                operator: In
                values:
                  - tx-aus
                  - tx-dal
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: disk-speed
                operator: In
                values:
                  - fast
                  - quick
```

- The `requiredDuringSchedulingIgnoredDuringExecution` rule ensures the Pod is only
  scheduled on nodes labeled `kubernetes.io/colo-tx-name` with values `tx-aus` OR
  `tx-dal`.

- The `preferredDuringSchedulingIgnoredDuringExecution` rule adds a preference for
  nodes labeled `disk-speed=fast` OR `disk-speed=quick`, giving them higher priority
  when available.

---

## Pod Affinity

Sama seperti Node Affinity, Pod affinity didefinisikan pada **Pod Spec**. Pod Affinity
merupakan mekanisme untuk menempatkan Pod agar **dekat** dengan Pods yang ada (Pods
-> Pods).

> **❓Dekat dari apa?**
> Terdapat field `topologyKey` yang digunakan untuk mendefinisikan 'dekat'.
> Misalnya:
>
> ```txt
> kubernetes.io/hostname        →  same node
> topology.kubernetes.io/zone   →  same availability zone
> topology.kubernetes.io/region →  same region
> ```

Pod Affinity juga memiliki [rules yang mirip dengan Node Affinity](#rules).

Contoh real case dari Pod Affinity:

```txt
# Case 1
App + its Redis cache
  app reads cache hundreds of times per request
  microsecond latency matters
  → podAffinity: schedule app on same node as redis
  → they communicate via localhost or node-internal network
  → avoids network hop between nodes


# Case 2
MySQL needs to stay on Node1 (using PV hostPath on Node1, so data is there):
  → nodeAffinity on MySQL pod: "only schedule on node1"
  OR better: use PVC with RWO instead of hostPath
             let scheduler place MySQL anywhere
             volume follows the pod automatically
```

### Example Pod Affinity

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: lab10-2-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lab10-2
  template:
    metadata:
      labels:
        app: lab10-2
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - hdd
      containers:
        - name: username-container
          image: busybox
          command: ["sleep", "3600"]
```

Pod diatas mendefinisikan agar Pods di-schedule HANYA pada Node yang memiliki label
dengan key `disktype` dengan nilai `hdd`.

> [!NOTE]
> Untuk mengetahui detail behaviour-nya:
>
> - Labeli salah satu Node dengan `disktype=hdd`
> - Delete pod lab10-2-pod-affinity
> - Change label nodes to disktype=ssd
> - Apply pod lab10-2-pod-affinity configuration
> - Check the status of the pod lab10-2-pod-affinity, is it in a pending or running
>   state?
> - If the pod lab10-2-pod-affinity is in a pending status, try changing the node
>   label back to disktype=hdd
> - If the disktype=hdd node label has been restored, the pod will transition to
>   a 'running' status

---

## Pod Anti-Affinity

Berkebalikan dengan [Pod Affinity](#pod-affinity), Pod Anti-Affinity merupakan mekanisme
untuk menempatkan Pod agar **jauh/berbeda** dengan Pods yang ada (Pods -> Pods).
Umumnya digunakan untuk high-availability dan vault-tolerance, sehingga Pods di
distribusikan ke Node lain secara merata.

> **❓Jauh/berbeda dari apa?**
> Terdapat field `topologyKey` yang digunakan untuk mendefinisikan 'jauh/berbeda'.
> Misalnya:
>
> ```txt
> kubernetes.io/hostname        →  same node
> topology.kubernetes.io/zone   →  same availability zone
> topology.kubernetes.io/region →  same region
> ```

Pod Anti-Affinity sama seperti Pod Affinity, yaitu didefinisikan di **Pod Spec**
dan mendukung rules dan operator yang sama seperti Pod Affinity dan Node Affinity.

### Example Pod Anti-Affinity

Berikut contoh Pod Anti-Affinity yang digunakan agar Pod aplikasi backend tidak
disimpan di Node yang sama. Sedangkan rules podAffinity yang digunakan yaitu agar
Pod aplikasi selalu berada/disimpan pada pod `cache`

```yaml
spec:
  affinity:
    # Pod AFFINITY — schedule near pods matching this selector
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache # schedule THIS pod on same node as cache pods
          topologyKey: kubernetes.io/hostname

    # Pod ANTI-AFFINITY — schedule AWAY from pods matching this selector
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: backend # don't put two backend pods on same node
          topologyKey: kubernetes.io/hostname
```

---

## Taints

Taint digunakan pada Node. Fungsinya repel / menolak Pod pada level Node (Node
-> Pod). Taints di ekspersikan dengan `key=value:effect`. Taints akan selalu
menolak (tergantung effect), kecuali Pod tersebut memiliki [toleration](#toleration).

Effect yang umum:

- `NoSchedule`. Scheduler tidak akan me-schedule Pod pada Node ini, kecuali Pod
  memiliki [toleration](#toleration). Pod yang sudah ada akan tetap berjalan, terlepas
  dari status toleration-nya.
- `PreferNoSchedule`. Scheduler akan menhindari node ini, kecuali jika tidak ada
  untainted nodes for the Pods toleration. Pod yang sudah ada tidak akan terpengaruhi
- `NoExecute`. Taint ini akan membuat Pod yang saat ini ada akan dihapus dan tidak
  akan ada Pod yang di-schedule pada Node ini. Jika sebuah Pod memiliki [toleration](#toleration)
  maka akan lanjut berjalan.

Contoh studi kasus yang umum yaitu:

```txt
Labels (grouping only):
  node has label gpu=true
  pods can choose to go there via nodeAffinity
  but nothing STOPS regular pods from also landing there
  regular pods just ignore the label

Taints (active repelling):
  node has taint dedicated=gpu:NoSchedule
  regular pods CANNOT land here even if they wanted to
  only pods with matching toleration can land here

---

Label alone:
  GPU node: [gpu-pod] [web-pod] [batch-pod]  ← everyone lands here
  regular nodes: [web-pod] [batch-pod]

Taint + Toleration:
  GPU node: [gpu-pod]   ← only pods with toleration
  regular nodes: [web-pod] [batch-pod]  ← everyone else goes here

---

Taint makes more sense for:
  expensive/special hardware you want to RESERVE exclusively
  GPU nodes: you don't want regular pods accidentally wasting GPU node resources
  spot/preemptible nodes: you want only fault-tolerant pods there
```

### Example Taints

1. Beri label pada sebuah Node dengan taint

   Misalnya memberikan label dengan taints `special=true:NoSchedule` pada node
   `pod-username-worker1`

   ```yaml
   kubectl taint nodes pod-username-worker1 special=true:NoSchedule
   ```

2. Buat simple Deployment

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: lab10-3-deployment
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
           - name: lab10-3-container
             image: nginx:latest
   ```

   Kemudian apply/deploy deployment ini

3. Verify Deployment. Maka Deployment akan ada di Node selain `pod-username-worker1`

   ```sh
   kubectl get deployment
   kubectl get pod -o wide
   ```

---

## Toleration

Toleration digunakan untuk melakukan bypass pada Node [Taints](#taints). Hal ini
memberikan cara yang mudah untuk menghidndari Pod menggunakan suatu node.
Hanya Pod yang memiliki toleration tertentu yang akan di-schedule.

> [!NOTE]
> Toleration saja tidak akan menempatkan Pod yang dipilih pada Node dengan Taints.
> Toleration hanya menghilangkan penghalang/rules (repel barrier)
>
> Just toleration:
>
> - pod CAN land on tainted node
> - but scheduler might still put it elsewhere if other nodes are better
> - pod floats wherever scheduler decides
>
> Untuk memastikan agar Scheduler aktif mencari Node, gunakan kombinasi Toleration
> \+ nodeAffinity.

Operator yang di dukung pada Toleration yaitu `Equal` (default), `Exists`.
Contoh:

```yaml
tolerations:
  - key: "server"
    operator: "Equal"
    value: "ap-east"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

### Example Toleration

> [!IMPORTANT]
> Pre-Requisite: Jalankan [example taints](#example-taints). Kemudian hapus Deployment
> sebelumnya.

1. Labeli Node dengan Taints

   Misalnya label Node dengan taints `error=true:PreferNoSchedule` pada Node
   `pod-username-worker2`

   ```sh
   kubectl taint nodes pod-username-worker2 error=true:PreferNoSchedule
   ```

2. Buat Deployment dengan Toleration

   Misalnya membuat Deployment agar Pod memiliki toleration pada label taints
   sebelumnya

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: lab10-3-deployment-v2
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         tolerations: # Toleration didefinisikan
           - key: "special"
             operator: "Equal"
             value: "true"
             effect: "NoSchedule"
         containers:
           - name: lab10-3-container-v2
             image: nginx:latest
   ```

   Kemudian apply/deploy Deployment ini

3. Verify Deployment. Maka seharusnya Pods dari Deployment akan hanya berjalan pada
   worker1

   ```sh
   kubectl get deploy
   kubectl get pods -l app=nginx -o wide
   ```
