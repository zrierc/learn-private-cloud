# Create Instance using Horizon Dashboard

- **Date:** July 3rd, 2026

## Overview

Catatan untuk mengakses dashboard OpenStack horizon tanpa akses publik (tidak memiliki
IP publik) / tidak di hosting.

### Topology

```txt
[my machine] ---> [baremetal] ---> [vm workstation @ 192.168.122.20]
                  (jump host) ---> [vm controller  @ 192.168.122.21]
                              ---> [vm compute1    @ 192.168.122.22]
                              ---> [vm compute2    @ 192.168.122.23]
```

Karena pada tahap [instalasi](./1-setup-intallation.md), menggunakan konfigurasi
`kolla_internal_vip_address: "192.168.122.100"`. Maka:

- Akses web dashboard Horizon: `http://192.168.122.100:80`
- Akses VNC Horizon (untuk console access setiap VM): `http://192.168.100:6080`

> [!TIP]
> Credetials untuk login dashboard tersedia pada `/etc/kolla/admin-openrc.sh` dengan
> environment variables`OS_USERNAME` dan `OS_PASSWORD`.

## Access trough SSH Tunnel

Untuk mengakses dashboard Horizon tanpa akses publik / tidak di hosting yaitu dengan
SSH Tunnel.

### Command Line

caranya yaitu:

```sh
ssh -L 8000:192.168.122.100:80 -J <jump-host-user>@<jump-host-ip> <user-vm-controller>@<ip-vm-controller> -N
```

details:

- Param `-L` untuk mengaktifkan tunnel
- Arg `8000:192.168.122.100:80` untuk mengakses web Horizon (dengan alamat `192.168.122.100:80`)
  di local machine port `8000`, sehingga aksesnya yaitu cukup dengan
  [localhost:8000](http://localhost:8000) di local machine.

- Params `-J` untuk mengakses VM controller melewati jump host.
- `<jump-host-user>@<jump-host-ip>` user dan alamat/domain **jump host**.
  Contoh: `rei@lab.nerv.com`
- `<user-vm-controller>@<ip-vm-controller>` user dan alamat VM. Contoh: `ubuntu@192.168.122.21`
- Params `-N` untuk menonaktifkan remote command (_do not execute a remote command_).
  Sangat direkomendasikan untuk penggunaan tunneling.

### Configuration File

#### 1. Setup Jump Host

Command pada [tahap sebelumnya](#command-line) cukup _straightforward_ tetapi terlalu
panjang. Command diatas dapat di sederhanakan dengan file konfigurasi.

Caranya edit `~/.ssh/config`

```config
Host <nama-bebas-asal-jangan-conflict>
  HostName <jump-host-ip-or-domain>
  User <jump-host-user>
```

Misalnya:

```config
Host bastion
  HostName lab.nerv.com
  User rei
```

> [!TIP]
> Nama `bastion` atau `bastion host` cukup umum untuk mendefinisikan sebuah server/vm
> yang digunakan hanya untuk jump host untuk alasan keamanan.

Jika koneksi menggunakan ssh key, dapat attach public key yang digunakan.

```config
Host bastion
  HostName lab.nerv.com
  User rei
  IdentityFile ~/.ssh/id_rsa.pub
```

#### 2. Setup Tunneling

Untuk tunneling-nya sendiri, perlu mendefinisikan `Host` baru. Caranya:

```config
Host bastion
  HostName lab.nerv.com
  User rei
  IdentityFile ~/.ssh/id_rsa.pub

Host <nama-bebas-untuk-alias-vm-controller>
  HostName <ip-vm-controller>
  User <user-vm-controller>
  IdentityFile </path/to/key.pub> # gunakan jika ada
  ProxyJump <nama-host-yang-digunakan-untuk-jump-host> # wajib ditambahkan
```

contoh:

```config
Host bastion
  HostName lab.nerv.com
  User rei
  IdentityFile ~/.ssh/id_rsa.pub

Host controller
  HostName 192.168.122.21
  User ubuntu
  IdentityFile ~/.ssh/key.pub # gunakan jika ada
  ProxyJump bastion # wajib ditambahkan
```

#### 3. (optional) Setup other VM alias

Bisa juga definisikan VM lain (misal `workstation`). Formatnya sama.
unuk
Contoh:

```config
Host bastion
  HostName lab.nerv.com
  User rei
  IdentityFile ~/.ssh/id_rsa.pub

Host controller
  HostName 192.168.122.21
  User ubuntu
  IdentityFile ~/.ssh/key.pub # gunakan jika ada
  ProxyJump bastion # wajib ditambahkan

Host workstation
  HostName 192.168.122.20
  User ubuntu
  IdentityFile ~/.ssh/key.pub # bisa sama
  ProxyJump bastion # wajib ditambahkan
```

#### 4. Start to open the Tunnel

Karena sudah di konfigurasi, maka [command awal](#command-line) bisa dipersingkat
menjadi:

```sh
ssh -L 8080:192.168.122.100:80 controller -N
```

Details:

- Param `-L` untuk mengaktifkan tunnel
- Arg `8000:192.168.122.100:80` untuk mengakses web Horizon (dengan alamat `192.168.122.100:80`)
  di local machine port `8000`, sehingga aksesnya yaitu cukup dengan
  [localhost:8000](http://localhost:8000) di local machine.

- Args `workstation` akan otomatis di _resolve_ berdasarkan config yang ada.
  Karena di dalam config sudah mendefinisikan `ProxyJump`, maka tidak perlu lagi
  menggunakan parameter `-J`.

- Params `-N` untuk menonaktifkan remote command (_do not execute a remote command_).
  Sangat direkomendasikan untuk penggunaan tunneling.

## Additional Tips: Nested SSH

> [!NOTE]
> Cocok untuk digunakan untuk SSH biasa (dengan interactive terminal)

[Step sebelumnya](#access-trough-ssh-tunnel) hanya bagus untuk tunneling. Command
itu juga bisa digunakan untuk SSH ke VM (seperti controller, workstation, dll)
tanpa harus manual akses ke jump-host. Caranya cukup **hilangkan parameter `-L`
dan `-N`**. Pastikan argumen pada parameter `-L` juga dihapus.

Contoh:

```sh
# ssh -J <jump-host-user>@<jump-host-ip> <user-vm-controller>@<ip-vm-controller>

ssh -J rei@lab.nerv.com ubuntu@192.168.122.21
```

Meskipun begitu, tetapi cara ini kurang efisien karena dapat meningkatkan
_input lag_ yang cukup tinggi dan mengurangi kenyamanan. Solusi alternatifnya
yaitu menggunakan **nested SSH**.

Caranya yaitu:

```sh
ssh -t <jump-host-user>@<jump-host-ip> 'ssh -t <user-vm-controller>@<ip-vm-controller>'
```

contoh:

```sh
ssh -t rei@lab.nerv.com 'ssh -t ubuntu@192.168.122.21'
```

details:

- params `-t` digunakan untuk membuka interactive terminal
- Mekanisme nya yaitu: ssh ke `rei@lab.nerv.com`, kemudian eksekusi perintah
  `ssh -t ubuntu@192.168.122.21` di dalamnya
