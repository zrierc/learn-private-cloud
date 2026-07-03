# Manage Authentication and Authorization using Project, User, Role, and Quota

- **Date:** July 3rd, 2026

## Overview

Belajar mengelola **Project**, **User**, **Role**, dan membatasi jumlah resources
yang tersedia di OpenStack dengan **Quota**.

Topologi lihat di [bagian sebelumnya](./1-setup-intallation.md).

### Pre-Requisite

- Pastikan OpenStack CLI telah terinstall
- Pastikan menggunakan file `openrc` yang sesuai untuk autentikasi ke OpenStack

---

## Manage project

### 1. List Project

```sh
openstack project list
```

### 2. Create Project

Membuat project `myproject-username` dengan deskripsi `my own project`.

```sh
openstack project create --description 'my own project' myproject-username --domain default
```

### 3. Show Detail Project

Lihat detail project `myproject-username`

```sh
openstack project show myproject-username
```

### 4. Disable Project (temporary)

Disable project `myproject-username`

```sh
openstack project set myproject-username --disable
```

### 5. Enable a Disabled Project

Enable kembali project yang sebelumnya sudah di disable (`myproject-username`):

```sh
openstack project set myproject-username --enable
```

### 6. Update Name of Project

Mengubah nama project `myproject-username` menjadi `mynewproject-username`

```sh
openstack project set myproject-username --name mynewproject-username
```

---

## Manage User

### 1. List User

```sh
openstack user list
```

### 2. Create User

Membuat user bernama `username` dengan password `YourStrongPassword` di project
`mynewproject-username`

```sh
openstack user create --project mynewproject-username --password "YourStrongPassword" username
```

### 3. Show Detail User

Lihat detail user bernama `username`

```sh
openstack user show username
```

### 4. Disable User

Disable user bernama `username`

```sh
openstack user set username --disable
```

### 5. Enable a Disabled User

Enable user yang sebelumnya sudah di disable (`username`)

```sh
openstack user set username --disable
```

### 6. Change User

Ubah nama dan email user bernama `username` menjadi nama `username-lab7.2` dan
email `username@mail.com`

```sh
openstack user set username --name username-lab7.2 --email username@mail.com
```

---

## Manage Role

### 1. List Role

```sh
openstack role list
```

### 2. Create Role

> [!NOTE]
> By default, terdapat role yang tersedia di OpenStack, yaitu:
>
> - `admin`: Full access. Dapat mengelola project, user, role, dan resources.
> - `member`: General purpose. Dapat digunakan untuk membuat, mengelola resources
>   di dalam project yang spesifik.
> - `reader`: Hanya untuk membaca resources yang tersedia di dalam project yang spesifik

Membuat role baru bernama `designer`

```sh
openstack role create designer
```

### 3. Assign User to Role

> [!NOTE]
> User di OpenStack support di assign dua atau lebih role

Memberi user `username-lab7.2` role `member` di dalam project `mynewproject-username`

```sh
openstack role add --user username-lab7.2 --project mynewproject-username member
```

### 4. List Assignment Role

Melihat detail role yang tertera pada user `username-lab7.2` pada
project `mynewproject-username`

```sh
openstack role assignment list --user username-lab7.2 --project mynewproject-username --names
```

### 5. Show Detail Role

Melihat detail role admin

```sh
openstack role show admin
```

---

## Manage Quota

> [!NOTE]
> Setiap Project memiliki quota untuk resources tertentu. Nilainya bervariasi.
> Tetapi yang unik yaitu terdapat nilai:
>
> - `-1`: menandakan bahwa resources yang berkaitan tidak memiliki quota limit
> - `0`: menandakan bahwa tidak diberikan akses ke resources tersebut

## 1. Show Limit in Project

Lihat detail quota pada project `mynewproject-username`

```sh
openstack quota show mynewproject-username
```

## 2. Update Quota

> [!TIP]
> Untuk lihat atau cari parameter untuk mengatur quota resources lain dapat
> menggunakan perintah:
>
> ```sh
> openstack quota set --help
> ```

Update quota untuk resources `instance` sebanyak 20 pada project `mynewproject-username`

```sh
openstack quota set --instance 20 mynewproject-username --force
```

> [!TIP]
> Setelah mengatur/update quota, cek kembali [limit-nya](#1-show-limit-in-project)
