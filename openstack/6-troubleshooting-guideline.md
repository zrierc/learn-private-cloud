# Troubleshooting OpenStack

- **Date:** July 6th, 2026

## Overview

Topologi lihat di [bagian sebelumnya](./1-setup-intallation.md).

Belajar melakukan troubleshooting pada OpenStack dari berbagai layer. Pattern
pendekatannya yaitu: cek service status, baca logs, cek network, dan trace file
konfigurasi yang berkaitan.

Sumber issue dari OpenStack terdiri dari beberapa layer:

- Infrastructure Layer (Host/Hypervisor)
  Examples: hardware failure, disk full, or physical network disconnected.

- Service Layer (OpenStack services)
  Examples: daemon dead, API service crash, or network agent not responding.

- Instance Layer (VM / user space)
  Examples: VM boot failure, unable to SSH, or unable to obtain an IP address.

- Configuration & Integration Layer
  Examples: errors in configuration files, unsynchronized credentials, or incorrect
  Keystone endpoints.

### Pre-Requisite

- Pastikan OpenStack CLI telah terinstall
- Pastikan menggunakan file `openrc` yang sesuai untuk autentikasi ke OpenStack

---

## Troubleshooting Steps

> [!TIP]
> Mulai troubleshooting dari node controller terlebih dahulu

### Check OpenStack Service Status

```sh
openstack service list
```

### Check Compute Service Status

Dapat digunakan juga untuk mengecek setiap node compute yang terhubung

```sh
openstack compute service list
```

### Check Resource Capacity

```sh
openstack hypervisor list
```

Untuk melihat detail setiap node nya:

```sh
openstack hypervisor show hostname
```

### Check Project Quota

```sh
openstack quota show project_name
```

### Check Service Log

Misalnya cek Nova compute log yang dijalankan di setiap compute node (misal
`compute1` dan `compute2`):

1. Cek log container yang menjalankan service Nova dari docker:

   ```sh
   sudo docker logs nova_compute | tail -30
   ```

2. Cek detail log

   ```sh
   sudo less /var/lib/docker/containers/*nova_compute*/nova-compute.log

   # lebih detail, cek nova_compute
   sudo less /var/lib/docker/containers/$(sudo docker ps -a --no-trunc | grep nova_compute | awk '{print $1}')/$(sudo docker ps -a --no-trunc | grep nova_compute | awk '{print $1}')-json.log
   ```

3. Cari error dari log

   ```sh
   grep ERROR /var/lib/docker/containers/*nova_compute*/nova-compute.log | tail -20

   # lebih detail, cek error pada nova_compute
   grep -i error /var/lib/docker/containers/$(sudo docker ps -a --no-trunc | grep nova-compute | awk '{print $1}')/$(sudo docker ps -a --no-trunc | grep nova-compute | awk '{print $1}')-json.log
   ```

---

## Troubleshooting Glance

1. Cek disk usage

   ```sh
   sudo df -h
   ```

   Jika `/var` atau `/var/lib/glance` hampir penuh, dapat hapus image lama yang
   tidak digunakan

2. Cek service status

   ```sh
   openstack image list
   ```

3. Cek container status

   ```sh
   sudo docker ps | grep glance
   ```

4. Lanjut ikuti [troubleshooting steps](#troubleshooting-steps). Sesuaikan nama service/dir-nya

## Troubleshooting Network

1. Cek status dari Neutron agent

   ```sh
   openstack network agent list
   ```

2. Cek container status

   ```sh
   sudo docker ps | grep neutron
   ```

3. (opsional, jika dibutuhkan) restart agent

   ```sh
   sudo docker restart neutron_l3_agent
   ```

4. Cek IP forwarding

   ```sh
   sudo sysctl net.ipv4.ip_forward
   ```

   Jika hasilnya `0`, maka aktifkan IP forwarding dengan

   ```sh
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

5. Verifikasi Bridge interface

   ```sh
   sudo ovs-vsctl show
   ```

6. Cek log docker neutron dan/atau lanjut ikuti [troubleshooting steps](#troubleshooting-steps).
   Sesuaikan nama service/dir-nya

## Troubleshooting Volume

1. Cek status service cinder:

   ```sh
   openstack volume service list
   ```

2. Cek log container

   ```sh
   sudo docker logs cinder_volume | tail -20
   ```

3. Cek kapasitas backend storage

   ```sh
   sudo df -h /var/lib/cinder
   ```

4. (opsional, jika dibutuhkan) restart container cinder

   ```sh
   sudo docker restart cinder_volume
   ```

5. Lanjut ikuti [troubleshooting steps](#troubleshooting-steps). Sesuaikan nama service/dir-nya

## Troubleshooting Dashboard Horizon/Keystone

1. Cek keystone status

   ```sh
   openstack endpoint list
   ```

2. Cek container service keystone

   ```sh
   sudo docker ps | grep keystone
   ```

3. Pastikan menggunakan credentials yang sesuai

   ```sh
   source admin-openrc.sh
   openstack project list
   ```

4. (jika NTP bermasalah) Lakukan time sync

   ```sh
   timedatectl status
   sudo systemctl restart chronyd
   ```

5. Cek log docker keystone dan/atau lanjut ikuti [troubleshooting steps](#troubleshooting-steps).
   Sesuaikan nama service/dir-nya
