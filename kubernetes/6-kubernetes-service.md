# Kubernetes Service

- **Date:** July 17th, 2026

## Overview

Eksplorasi dan belajar mengenai Service di Kubernetes

---

## Service

Jika mendeploy aplikasi di Kubernetes, secara default aplikasi hanya bisa diakses
didalam internal cluster Kubernetes. Agar bisa diakses dari luar, maka diperlukan
Kubernetes Object bernama Service.

Terdapat beberapa jenis Service

- `ClusterIP`: Merupakan tipe default service. Memberikan kemampukan akses internal.
  Service tidak bisa diakses dari luar

- `NodePort`: Service tipe ini membuka spesifik port pada setiap Node dalam cluster
  dan mengarahkannya ke setiap Pod melalui IP address node, caranya yaitu mengarahkan
  port yang di ekspose ke port internal yang ada di service. Umumnya port yang di
  ekspose ada di disekitar rentang 30.000 - 32.767. Sehingga dapat diakses dengan
  format `<NODE-IP>:<NodePort>`

- `LoadBalancer`: Service tipe ini melakukan distribusi traffic ke setiap Pods yang
  ada. Cara kerja dibelakangnya yaitu Kubernetes membuat `ClusterIP` dan `NodePort`
  service dan external Load Balancer kemudian routes traffic ke setiap pods.

Kubernetes component utama yang dibutuhkan service yaitu `kube-proxy`. Komponen ini
merupakan core network component yang berjalan disetiap node. Bertugas untuk mengelola
routing, dan memastikan bahwa traffic diarahkan ke service yang sesuai.

### Manage Service

Umumnya `Service` dibuat dan didefinisikan menggunakan file YAML. Meskipun begitu
service masih tetap dapat di dengan `kubectl`

#### Get Services

```sh
kubectl get services

# atau disingkat
kubectl get svc
```

#### Create ClusterIP Service

1. Buat Deployment

   Membuat bernama `lab8-1-deployment` dengan `3` replica dan memiliki image `nginx`
   yang berjalan di port `80` serta melabeli app nya dengan nama `nginx`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: lab8-1-deployment
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
           - name: lab8-1
             image: nginx:latest
             ports:
               - containerPort: 80
             volumeMounts:
               - name: custom-index
                 mountPath: /usr/share/nginx/html
         volumes:
           - name: custom-index
             configMap:
               name: lab-8-1-custom-index-configmap
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: lab-8-1-custom-index-configmap
   data:
     index.html: |
       username lab 8.1
   ```

   Kemudian deploy Deployment tersebut. Pastikan berhasil berjalan

   ```sh
   kubectl apply -f lab8-1-deployment.yaml
   kubectl get deploy
   ```

2. Buat ClusterIP untuk Deployment yang dibuat sebelumnya

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: lab-8-1-clusterip
   spec:
     type: ClusterIP
     selector:
       app: nginx
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
   ```

   Kemudian deploy Service tersebut

   ```sh
   kubectl apply -f lab8-1-svc.yaml
   kubectl get svc
   ```

#### Create NodePort Service

> [!NOTE]
> `NodePort` dapat diakses dari semua node yang ada di cluster. Baik itu master
> ataupun worker.

Membuat Service `NodePort` dari Deployment [sebelumnya](#create-clusterip-service).
Service memilih Deployment melalui label selector

```yaml
apiVersion: v1
kind: Service
metadata:
name: lab8-2-service-nodeport
spec:
selector:
    app: nginx
ports:
    - protocol: TCP
    port: 80
    targetPort: 80
type: NodePort
```

Kemudian deploy Service tersebut

```sh
kubectl apply -f lab8-2-svc.yaml
```

Untuk mengecek port yang dapat diakses dapat menggunakan perintah

```sh
kubectl get svc
```

atau untuk lebih spesifik melihat port pada service sebelumnya (bernama `lab8-2-service-nodeport`)

```sh
kubectl get svc | grep lab8-2-service-nodeport | cut -d "/" -f 1 | cut -d ":" -f 2
```

#### Deploy Multi-Tier Application

Studi kasus ini akan mendeploy MongoDB backend dan Nginx-based Front-End

1. Buat deployment MongoDB

   ```sh
   kubectl apply -f https://raw.githubusercontent.com/bta-adinusa/bta/main/rsvp-db.yaml
   ```

   Pastikan Deployment berhasil dan Pods berstatus `Running`. Lihat dengan:

   ```sh
   kubectl get deploy
   ```

2. Buat Service MongoDB

   Tipe `ClusterIP`

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mongodb
     labels:
       app: rsvpdb
   spec:
     ports:
       - port: 27017
         protocol: TCP
         targetPort: 27017
     selector:
       appdb: rsvpdb
   ```

   Kemudian deploy Service tersebut

   ```sh
   kubectl apply -f mongo-svc.yaml
   ```

   Pastikan Service berhasil. Lihat dengan:

   ```sh
   kubectl get svc
   ```

3. Buat deployment Front-End

   ```sh
   kubectl apply -f https://raw.githubusercontent.com/bta-adinusa/bta/main/rsvp-web.yaml
   ```

   Pastikan Deployment berhasil dan Pods berstatus `Running`. Lihat dengan:

   ```sh
   kubectl get deploy
   ```

4. Buat Service `NodePort` untuk Front-End

   Membuat Service bernama `rsvp` bertipe `NodePort` dengan menspesifikan port
   yang diekspose yaitu port `30036`

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: rsvp
     labels:
       apps: rsvp
   spec:
     type: NodePort
     ports:
       - port: 80
         targetPort: web-port
         nodePort: 30036
         protocol: TCP
     selector:
       app: rsvp
   ```

   Kemudian deploy Service tersebut

   ```sh
   kubectl apply -f web-svc.yaml
   ```

   Pastikan Service berhasil. Lihat dengan:

   ```sh
   kubectl get svc
   ```

5. Scale Front-End

   > [!NOTE]
   > `rsvp-web.yaml` adalah file external dari link pada saat apply Deployment
   > Front-End. Jika tidak tersedia, cukup download file nya (misal dengan `wget`)

   ```sh
   kubectl scale --replicas=3 -f rsvp-web.yaml
   ```

   Pastikan Deployment berhasil di scale. Lihat dengan:

   ```sh
   kubectl get deployments
   ```

6. Akses Website

   Akses melalui browser: `http://<master-node-ip>:30036/`

   atau melalui CLI:

   ```sh
   curl http://<master-node-ip>:30036/

   # misal
   curl http://10.1.1.10:30036/
   ```

#### Delete Service

```sh
kubectl delete -f /path/to/service.yaml
```

---

## Local Proxy

Ketika develop/testing aplikasi di Kubernetes biasanya membutuhkan akses/pengecekan
bahwa Pods/Service sudah berjalan sesuai yang diinginkan tanpa harus mengekspose
secara external. Salah satu solusinya yaitu dengan `kubectl proxy`

`kubectl proxy` akan membuat local gateway sementara (temporary) yang melakukan
autentikasi ke Kubernetes API dengan local credentials yang digunakan sehingga
bisa mengakses aplikasi/API endpoints serta megakses internal ClusterIP Service
melalui `localhost`

Untuk menjalankannya, jalankan perintah:

```sh
kubectl proxy
```

> [!NOTE]
> Outputnya akan menampilkan endpoint local beserta port yang dapat digunakan.
> Contoh:
>
> ```sh
> Starting to serve on 127.0.0.1:8001
> ```

Setelah berjalan, aplikasi dapat diakses dengan menambahkan suffix endpoint `/proxy/`

Misalnya mengakses service bernama `ghost` dengan `default` namespaces

```sh
http://localhost:8001/api/v1/namespaces/default/services/ghost/proxy/
```

Jika service mengekspose multiple port, maka dapat menspesifikan nama port dengan
format `/<port-name>:<service-name>:/proxy/`

Misalnya mengakses port bernama `web` di service bernama `ghost` dengan `default`
namespaces

```sh
http://localhost:8001/api/v1/namespaces/default/services/web:ghost:/proxy/
```

> [!IMPORTANT]
> Jika mengakses endpoint tanpa prefix `/proxy/`, maka URL akan me return konfigurasi
> JSON dari object Service itu sendiri

---

## DNS

In Kubernetes, Pods often need to communicate with other Pods or Services using
their names instead of dynamic IP addresses. This critical name resolution is handled
by `CoreDNS`, the cluster’s built-in, default DNS server. It’s a lightweight, modular,
and flexible system that automatically translates service names (like
`myapp.default.svc.cluster.local`) into the correct Pod IPs behind the scenes.

Untuk verify DNS di Kubernetes, dapat menggunakan perintah:

```sh
kubectl exec -it dns-test -- nslookup kubernetes.default
```

If DNS is configured correctly, the service name will successfully resolve to its
ClusterIP address.
