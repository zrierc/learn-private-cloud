# Manage Block Storage using Cinder

- **Date:** July 6th, 2026

## Overview

Belajar mengelola persistent block storage dengan Cinder untuk Virtual Hard drives.

Topologi lihat di [bagian sebelumnya](./1-setup-intallation.md).

### Pre-Requisite

- Pastikan OpenStack CLI telah terinstall
- Pastikan menggunakan file `openrc` yang sesuai untuk autentikasi ke OpenStack

---

## Manage Volumes

### 1. List Volumes

```sh
openstack volume list
```

### 2. Create Volumes

Membuat volume sebesar 5GB dengan nama `myvolume-username`

```sh
openstack volume create --size 5 myvolume-username
```

### 3. Show Detail Volumes

- Melihat detail volume yang telah dibuat, bernama `myvolume-username`

```sh
openstack volume show myvolume-username
```

- Untuk mengecek status ketersediaan volume, misalnya mengecek volume `myvolume-username`

```sh
openstack volume show myvolume-username -f value -c status
```

> [!NOTE]
> Hasilnya akan ada:
>
> - `available`: Tersedia dan dapat digunakan/di attach
> - `in-use`: Tersedia tetapi sedang digunakan
> - `creating`: Sedang dibuat, belum tersedia
> - `deleting`: Sedang dihapus, tidak tersedia
> - `error`: Tidak tersedia, terdapat kegagalan pada pembuatan/pemasangan (attaching)/
>   snapshot
> - `migrating`: Belum tersedia, volume sedang dipindahkan antara backend nodes

### 4. Attach Volumes

Memasang (attach) volume `myvolume-username` ke instance bernama `cirros-instance-username`
di `/dev/vdb`

```sh
openstack server add volume cirros-instance-username myvolume-username --device /dev/vdb
```

#### Verify

1. SSH ke instance `cirros-instance-username`

2. Cek volume

   ```sh
   sudo lsblk
   ```

> [!TIP]
> Pastikan output `lsblk` menampilkan volume `/dev/vdb`

### 5. Detach Volumes

Untuk melepas (detach) volume `myvolume-username` dari instance bernama `cirros-instance-username`

```sh
openstack server remove volume cirros-instance-username myvolume-username
```

---

## Snapshot and Restore

Setiap volume yang dibuat dapat di backup dan di restore. Mekanisme ini menggunakan
snapshot.

### 1. List Snapshot

#### Snapshot Instance (Image)

Men-list/cari image hasil snapshot bernama `snap-vm-lab`

```sh
openstack image list --name snap-vm-lab
```

#### Snapshot Volume

```sh
openstack volume snapshot list
```

### 2. Take a Snapshot of an Instance (Image)

> [!IMPORTANT]
> Snapshot instance akan membuat image baru dari state instance disk saat ini

Membuat snapshot dari instance `vm-rescue` bernama `snap-vm-lab`

```sh
openstack server image create --name snap-vm-lab vm-rescue
```

Cek status pembuatannya

```sh
openstack image list | grep snap-vm-lab
```

> [!TIP]
> Tunggu hingga status menjadi `ACTIVE`

### 3. Take a Snapshot of a Volume

Membuat snapshot / backup bernama `snap-vol-lab` dari volume bernama `myvolume-username`

```sh
openstack volume snapshot create --volume myvolume-username snap-vol-lab
```

#### check status

Cek status pembuatan snapshot sebelumnya

```sh
openstack volume snapshot show snap-vol-lab -f value -c status
```

> [!TIP]
> Pastikan statusnya berubah dari `creating` menjadi `available`

### 5. Restore

Untuk melakukan restore dapat me-attach hasil snapshot ke instance.

#### Image

Misalnya membuat instance baru bernama `vm-restored` dari snapshot image

```sh
NET_ID=$(openstack network list -f value -c ID -c Name | grep internal-net | grep horizon | awk '{print $1}')
FLAVOR=$(openstack flavor list -f value -c Name | grep c1-small | head -n1)

openstack server create \
  --flavor "$FLAVOR" \
  --image snap-vm-lab \
  --nic net-id="$NET_ID" \
  --key-name controller-key-horizon \
  vm-restored
```

> [!TIP]
> Pastikan cek status pembuatan VM secara berkala hingga status-nya `ACTIVE`.
> Untuk cek status VM:
>
> ```sh
> openstack server show vm-restored -c status
> ```

#### Volume

Kemudian untuk restore volume, buat volume baru dari snapshot, kemudian attach
ke instance bernama `vm-restored`

```sh
openstack volume create --snapshot snap-vol-lab vol-restore
```

> [!TIP]
> Pastikan cek status pembuatan volume secara berkala hingga status-nya `available`.
> Untuk cek status volume:
>
> ```sh
> openstack volume show vol-restore
> ```

Jika volume sudah `available`, pasang (attach) volume hasil snapshot (bernama `vol-restore`)
ke instance `vm-restored`

```sh
openstack server add volume vm-restored vol-restore
```

---

## Troubleshooting

> [!NOTE]
> Untuk lebih detail dapat cek disini:
>
> - [Volume error](./findings/host-restart-and-missing-virtual-disks.md)
> - [Network/routing error](./findings/host-restart-and-network-interface-down.md)

Kondisi ini terjadi ketika hosts (controller / compute1 / compute2) mati. Kemudian
ketika host dijalankan kembali, virtual disk yang digunakan untuk OpenStack Cinder
menghilang. Pada studi kasus ini virutal disk nya yaitu `/dev/vdb`.

Jika terjadi hal seperti ini, status volume akan bernilai `error`. Untuk mengatasinya:

1. Attach kembali virtual disk `/dev/vdb`

2. Verify bahwa volume sudah kembali

   Check block device, pastikan `/dev/vdb` tersedia

   ```sh
   sudo lsblk
   ```

   Check LVM, pastikan ada `cinder-volumes`

   ```sh
   sudo vgs
   ```

3. Restart cinder container di semua host

   > [!TIP]
   > Setelah di restart, tunggu hingga service healthy. Mungkin memakan waktu sekitar
   > 30-60 detik. Untuk cek service:
   >
   > ```sh
   > docker ps | grep cinder
   > ```

   ```sh
   # Controller
   sudo docker restart cinder_volume cinder_scheduler cinder_backup

   # Compute nodes
   ssh ubuntu@pod-username-compute1 "sudo docker restart cinder_volume cinder_backup"
   ssh ubuntu@pod-username-compute2 "sudo docker restart cinder_volume cinder_backup"
   ```

4. Pastikan cinder service berjalan

   > [!TIP]
   > Service berjalan akan memiliki status `UP`

   ```sh
   openstack volume service list
   ```

5. Recovery volume yang error

Terdapat 2 cara untuk recovery:

a. Reset State

> [!TIP]
> Gunakan ini jika volume sudah berisi data. Misalnya data penting

```bash
openstack volume set --state available <volume-name>
```

b. Buat ulang

> [!TIP]
> Sangat direkomendasikan. Gunakan ini jika volume baru dibuat

```bash
openstack volume delete <volume-name>
openstack volume create --size <size-gb> <volume-name>
```
