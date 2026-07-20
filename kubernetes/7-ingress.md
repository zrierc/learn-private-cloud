# Kubernetes Ingress

- **Date:** July 20th, 2026

## Overview

Eksplorasi dan belajar mengenai Ingress di Kubernetes

---

## Ingress

Ingress digunakan untuk mengekspose app ke external network dengan HTTP atau HTTPS.
Ingress digunakan sebagai gateway untuk me-route traffic.

> ❓**But why not just Service?**
> Service dengan tipe `NodePort` me-expose port. Umumnya port 30000 - 32767.
> Tidak user friendly. </br>
> Service dengan tipe`LoadBalancer` hanya merepresentasikan satu service saja.
> Jika ada 10 service maka perlu: 10 konfigurasi `LoadBalancer`, 10 SSL certificate,
> setiap LoadBalancer memakan resource yang cukup besar. Tidak efisien

Ingress memungkinkan untuk menjalankan satu komponen komponen saja, yaitu
Ingress Controller. Dapat di distribusikan atau mengelola banyak services sekaligus.

Terdapat 2 istilah yang merujuk ke `ingress`, yaitu:

- Ingress Object: konfigurasi berisi routing rules dengan format YAML
- Ingress Controller: Actual pods yang menjalankan dan melakukan routing traffic
  ke service(s).

> [!IMPORTANT]
> Ingress tetap membutuhkan Service (misal dengan tipe `ClientIP`) dan Deployment
> agar bisa me-route traffic. Ingress tidak bisa langsung mengakses Deployment

Ingress sendiri adalah sebuah plugin (mirip seperti CNI, atau CSI), maka terdapat
beberapa opsi yang dapat digunakan, misalnya:

- [ingress-nginx](https://github.com/kubernetes/ingress-nginx) (berbasis nginx,
  cocok untuk belajar, tetapi sudah deprecated semenjak Maret, 2026. Alternatifnya
  [Gateway API](https://gateway-api.sigs.k8s.io/guides/)
- Traefik
- HAProxy Ingress
- Kong Ingress
- dll

---

## Manage Ingress

> [!NOTE]
> Umumnya Ingress dikelola dengan file YAML. Tetapi masih memungkinkan untuk
> mengelolanya dengan `kubectl`

### Get Ingress

```sh
kubectl get ingress
```

### Deploy Plugin Ingress Controller

Men-deploy Ingress Controller `ingress-nginx`

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

Jika sudah berhasil menginstall, verify dengan:

```sh
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Pastikan service ingress-nginx tersedia dan pods ber-status `Running`

### Create Ingress Rule

> [!NOTE]
> Service harus tersedia. Asumsi service telah dibuat

#### Single Rules

> [!TIP]
> Ingress akan mengarahkan traffic ke service port. Bukan `NodePort`

Mendefinisikan ingress rule bernama `myapp-ingress` yang mengarahkan path `/` pada
domain `myapp.com` ke service `web-svc` yang berjalan di port `8000`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx # which ingress controller handles this
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc # which service
                port:
                  number: 8000
```

#### Multiple Host Rules

Mendefinisikan ingress rule dengan dua host berbeda:

- `web-nginx.ok` -> service `web-nginx`
- `web-apache.ok` -> service `web-apache`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - host: web-nginx.ok
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-nginx
                port:
                  number: 80
    - host: web-apache.ok
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-apache
                port:
                  number: 80
```

### Multiple Path and Host Rules with SSL

- `https://myapp.com/` -> service `frontend-svc`
- `https://myapp.com/api/` -> service `backend-svc`
- `https://admin.myapp.com/` -> service `admin-svc`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx # which ingress controller handles this
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls-cert # SSL cert stored as Secret
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 3000
    - host: admin.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 9000
```

### Edit Ingress

jika melalui `kubectl`, maka:

```sh
kubectl edit ingress <nama-igress>
```

### Delete Ingress

```sh
kubectl delete -f /path/to/ingress-rules.yaml
```

jika menghapus langsung, maka:

```sh
kubectl delete ingress <nama-igress>
```
