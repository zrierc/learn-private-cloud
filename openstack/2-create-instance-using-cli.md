# Create Instance(s) using CLI

- **Date:** July 2nd, 2026

## Overview

Mencoba membuat beberapa resources (network dan compute) di OpenStack menggunakan
CLI.

Topologi lihat di [bagian sebelumnya](./1-setup-intallation.md)

Format atau 'rumus' menggunakan OpenStack CLI:

```sh
openstack <nama-resource> <operasi> [opsional --<params>] <arg>

# Contoh
# openstack image list
# openstack router create my-router
```

### Pre-Requisite

- OpenStack CLI

### Additional Setup

OpenStack yang disetup menggunakan Kolla Ansible memiliki RC file. RC file ini
berisi credentials yang bisa digunakan untuk melakukan autentikasi ke OpenStack
(baik melalui CLI ataupun dashboard Horizon).

RC file umumnya ada di `/etc/kolla/admin-openrc.sh` dan dibuat saat tahapan
Post-Deployment pada instalasi.

```sh
cat /etc/kolla/admin-openrc.sh
```

Contoh isi file:

```sh
for key in $( set | awk '{FS="="} /^OS_/ {print $1}' ); do unset $key ; done
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=
export OS_AUTH_URL=http://10.XX.XX.100:35357/v3
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE=internalURL
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
```

1. Load RC file ini agar resources di OpenStack dapat diakses dan dikola

   ```sh
   source /etc/kolla/admin-openrc.sh
   ```

2. (opsional) Aktifkan Virtual Environment Python (venv) jika install OpenStack CLI
   didalam nya

   ```sh
   source ~/kolla-venv/bin/activate
   ```

---

## Image

### 1. Download Image

> [!TIP]
> Download image dengan format `qcow2` atau `img`. Pastikan arsitektur-nya sesuai
> (misal `x86-64/amd64` atau `arm64` atau lainnya)

Pada praktik ini, men-download CirrOS

```sh
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
```

Bisa juga download image lain seperti CentOS, Ubuntu, dll.

```sh
# Ubuntu 20.04
wget https://cloud-images.ubuntu.com/releases/focal/release/ubuntu-20.04-server-cloudimg-amd64.img

# Ubuntu 24.04
wget https://cloud-images.ubuntu.com/releases/noble/release/ubuntu-24.04-server-cloudimg-amd64.img

# Ubuntu 26.04
wget https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img

# CentOS 10
wget https://cloud.centos.org/centos/10-stream/x86_64/images/CentOS-Stream-GenericCloud-10-latest.x86_64.qcow2
```

### 2. Create Image

```sh
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --file ./cirros-0.6.2-x86_64-disk.img \
  cirros-0.6.2
```

### 3. List Image

```sh
openstack image list
```

---

## External Network

> [!IMPORTANT]
> Meski OpenStack di deploy di containers, seluruh services (misalnya Neutron untuk
> network & subnet) tetap dibuat di atas Host menggunakan OVN/OVS, bukan containers.
> Neutron hanya membuat/menulis route traffic ke OVN/OVS.
>
> Misalnya:
> _Neutron API write rules to OVN/OVS:
> "VM-A (MAC: fa:16:3e:xx) lives on compute-01, VLAN 100"
> "Floating IP 203.0.113.45 → DNAT → 10.0.1.10"
> "Security group: allow TCP port 80 from 0.0.0.0/0"_

### External Subnet

---

## Internal Network

### Internal Subnet

---

## Router

---

## NIC / Port

> [!CAUTION]
> Saat create instance, jika sudah menggunakan `--port` untuk custom IP, maka tidak
> perlu ditambahkan `--network`. Hal ini akan menyebabkan MAC Address conflict/mismatch
> sehingga menyebabkan instance tidak bisa diakses (misal `ping` atau `ssh`)

---

## Security Group

---

## Keypair (import existing)

### 0. Pre-Requisite: Create Keypair

Buat SSH key dengan tipe RSA

```sh
ssh-keygen -t rsa -b 4096 -C "rei@lab.nerv.com"
```

Maka akan dibuat otomatis SSH key (sepasang private dan public key) di `~/.ssh`
dengan nama `id_rsa` (private key) dan `id_rsa.pub` (public key)

### 1. Import Public Key

Membuat keypair bernama `controller-key` dengan mengimpor public key yang sudah
dibuat sebelumnya

```sh
openstack keypair create --public-key ~/.ssh/id_rsa.pub controller-key
```

### 2. List keypair

```sh
openstack keypair list
```

### 3. Show Detail keypair

Lihat detail keypair `controller-key`

```sh
openstack keypair show controller-key
```

---

## Compute

Compute service di OpenStack bernama `Nova`.

> [!IMPORTANT]
> Meski OpenStack di deploy di containers, seluruh services (misalnya Nova untuk
> VM) tetap dibuat di atas Host, bukan containers. Sehingga flow-nya:
>
> ```txt
> [host (vm/baremetal)] --> [OpenStack containers] (NOVA do some API call to libvirt/KVM)
> [host (vm/baremetal)] --> [VM created by OpenStack]
> ```

### Flavor

### Instance

### Floating IP

---
