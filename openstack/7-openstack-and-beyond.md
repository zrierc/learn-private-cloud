# Temuan Saat Mengerjakan Lab Challenge OpenStack

- **Date:** July 7th, 2026

## Overview

Topologi lihat di [bagian sebelumnya](./1-setup-intallation.md).

Catatan pribadi berisi temuan-temuan/hal menarik saat mengerjakan challenge lab
(lab akhir) OpenStack.

---

## New Routes, where's the street sign?

### Overview Issues

Hal ini terjadi karena network interface (misal `ens4`) yang digunakan untuk external
network pada host (`controller` node) belum memiliki IP ATAU `external network`
pada Neutron OpenStack menggunakan IP fiktif (Misal `20.1.1.0/24`, tetapi IP public
IP itu tidak dimiliki atau tidak terhubung ke `ens4`).

### FAQ

#### 1

Q: Jika external network memiliki range yang sama dengan host network interface
(misal `ens4`), apakah masalah ini tetap ada?
A: Tidak. Misalnya jika setup nya:

```txt
ens4: 203.0.113.1/24  (real public IP)
external network: 203.0.113.0/24
external gateway: 203.0.113.1
```

Maka Neutron akan mengatur default route ke `203.0.113.1`. Karena ggateway tersebut
benar-benar ada, maka packet/network akan secara natural di route ke internet.

#### 2

Q: Bagaimana jika network interface `ens4` tidak memiliki IP/range IP?
A: Sama seperti kasus IP fiktif. Hal ini memerlukan manual routing:

- Manual assign IP external network (misal `20.168.168.1`) ke `br-ex` (sebuah external
  bridge yang dibuat oleh OVS (Open vSwitch) untuk menghubungkan network virtual
  ke infra fisik/nyata)
- Ganti gateway fiktif ke IP yang di assign sebelumnya (`20.168.168.1`)
- setup NAT ke internal network OpenStack (misalnya host menggunakan `ens3` untuk
  internal network)

#### 3

Q: Kapan perlu melakukan manual routing?
A: Setiap:

- Membuat router baru + set external gateway (Perlu fix default route `ip route replace`)
- Membuat internal subnet baru + menambahkan subnet the router yang sudah ada
  (Perlu tambahkan `MASQUERADE` rule baru untuk subnet tersebut)
- Host `controller` di reboot (Semuanya reset -- perlu perbaikan full, dapat diatasi
  di `rc.local` file)

### Step Fixing

Check routes dan nama interface nya:

```sh
sudo ip netns exec qrouter-<router-id> ip route show
```

Contoh output:

```txt
default via 20.168.168.2 dev qg-219a4eeb-51
20.168.168.0/24 dev qg-219a4eeb-51 proto kernel scope link src 20.168.168.162
192.168.88.0/24 dev qr-4d1efc03-4f proto kernel scope link src 192.168.88.1
```

> [!TIP]
> Simpan/catatat ID interface: `dev qg-219a4eeb-51`.

#### 1. Fix QRouter Default Route

```sh
sudo ip netns exec qrouter-<router-id> \
  ip route replace default via <external-ip-gateway> <id-interface>


# contoh
sudo ip netns exec qrouter-7c0b8cf5-5816-4f2d-af34-02b760a8dd1e \
  ip route replace default via 20.168.168.2 dev qg-219a4eeb-51
```

#### 2. Add MASQUERADE for new Subnet

```sh
sudo iptables -t nat -A POSTROUTING -s <internal-subnet-range> \
  -o <network-interface-host-untuk-internal-network-openstack> \
  -j MASQUERADE

# Contoh
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o ens3 -j MASQUERADE
```

#### 3. Verify

> [!TIP]
> Pastikan outputnya menampilkan IP external IP Gateway dan/atau range IP internal
> subnet yang dikonfigurasi

- Check jika qrouter sudah sesuai

  ```sh
  sudo ip netns exec qrouter-<router-id> ip route show

  # Contoh
  sudo ip netns exec qrouter-7c0b8cf5-5816-4f2d-af34-02b760a8dd1e ip route show
  ```

- Check MASQUERADE rules

  ```sh
  sudo iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE
  ```

- (opsional) Check internet di dalam VM/instance

  ```sh
  ping -c 3 1.1.1.1
  ping -c 3 google.com
  ```

---

## What is Cloud Init anyway?

### Cloud Init Overview

OpenStack Nova menyediakan fitur Cloud Init, yaitu sebuah fitur yang menyediakan
setup instances/VM saat membuatnya sehingga proses konfigurasi dapat dilakukan lebih
cepat dan otomatis. Baca lebih lanjut tentang Cloud Init [disini](https://docs.cloud-init.io/en/latest/).

Fitur ini bernama userdata, dan dapat menggunakan bash script ataupun
[template cloud init](https://docs.cloud-init.io/en/latest/reference/modules.html)
dengan format yaml.

### How

Cara menggunakan/menambahkan Cloud Init yaitu dapat membuat file bash script atau
membuat template cloud init dengan format yaml. File ini nantinya dapat di copy/paste
atau di upload langsung saat pembuatan instance.

#### 1. CLI

Untuk menggunakan Cloud Init via CLI, cukup gunakan params `--userdata` saat membuat
(`openstack server create`) instance. Argumen dari params tersebut dapat diisi dengan
path menuju file cloud init / bash script. Dalam contoh ini file bernama `cloud-init.yaml`.

```sh
openstack server create \
  ...
  --user-data /path/to/cloud-init.yaml \ # dapat diganti dengan file bash script
  <server-name>

# Example:
openstack server create \
  --flavor m1.small \
  --image "Ubuntu 24.04 LTS" \
  --network private-net \
  --key-name rei-nerv-key \
  --security-group rei-sg \
  --user-data /path/to/cloud-init.yaml \ # dapat diganti dengan file bash script
  server-rei
```

#### 2. Horizon Web Dashboard

---

## Live Migration on the Edge

### Live Migration Overview

OpenStack Nova menyediakan migration instance/VM untuk dipindahkan ke `compute` node
lain, yang menarik adalah adanya [Live Migration](https://wiki.openstack.org/wiki/Nova/Live_Migration).
Live migration ini menawarkan [minimal / near-zero downtime](https://wiki.openstack.org/wiki/Upgrade-with-minimal-downtime#Live_migration)
saat dimigrasikan.

### Pre-Migration

> [!NOTE]
> Digunakan saat terjadi ERROR instance

1. Cek status instance yang akan dimigrasikano

   ```sh
   openstack server list
   ```

   Jika status server yang akan dimigrasikan `ACTIVE`, dapat lanjut ke [tahap berikutnya](#how-to-migrate)

2. Jika status server `ERROR`

Hal ini dikarenakan terjadi masalah. Lakukan pengecekan root causes. Lihat detail
[di dokumen sebelumnya](./6-troubleshooting-guideline.md). Setelah itu, untuk instance/VM
yang `ERROR` terdapat 2 solusi:

- Hard Reboot

  ```sh
  openstack server reboot --hard <instance-id>
  ```

- Rebuild VM

  > [!CAUTION]
  > Tahap ini akan menjalankan ulang custom user data (cloud init) dan menghapus
  > semua data yang ada di ephemeral storage (storage bawaan).
  > Backup data terlebih dahulu ke volume atau buat snapshot.

  ```sh
  openstack server rebuild <instance-id> --image <image-name-or-id>
  ```

3. Cek status server

   ```sh
   openstack server list
   ```

   > [!TIP]
   > Server yang di rebuild akan memiliki status `REBUILD`. Tunggu hingga `ACTIVE`.

### How to Migrate

1. Cek terlebih dahulu [pre-migration](#pre-migration)

2. Cek host (compute node) yang menjalankan instance/VM yang akan dimigrasikan

   ```sh
   openstack server show <instance-id> | grep host
   ```

   > [!NOTE]
   > Akan menampilkan `hostId`, `host_status`, `OS-EXT-SRV-ATTR:host`, `OS-EXT-SRV-ATTR:hypervisor_hostname`
   > yang berisi host yang menyimpan/menjalankan VM/instance saat ini.

3. Jalankan migrasi

   > [!NOTE]
   > Live migration dapat dilakukan dengan params `--live-migration` dan status instance/VM
   > harus `ACTIVE`. Jika tidak menggunakan params, maka akan melakukan cold migration
   > yang hanya bisa dilakukan pada instance/VM yang memiliki status `SHUTOFF`.

   ```sh
   openstack server migrate --live-migration --host <destination-host> <instance>
   ```

   > [!TIP]
   > Pastikan `<destination-host>` tidak sama dengan host saat ini. Jika tidak migrasi
   > akan error

4. Cek status migrasi

   > [!NOTE]
   > Akan menampilkan status migrasi yaitu: `preparing`, `migrating`, `post-migrating`,
   > `finished`, dan `error`

   ```sh
   openstack server migration list --server <instance>
   ```
