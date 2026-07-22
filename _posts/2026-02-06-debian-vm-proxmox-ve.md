---
title: Debian VM с cloudinit в Proxmox VE (CLI)
date: 2026-02-06 8:00:00 +0700
categories: [Linux, How-tos]
tags: [openwrt, linux, proxmox, vm]     # TAG names should always be lowercase
---

Скачиваем образ
```bash
wget -O debian-13-backports-genericcloud-amd64.qcow2 https://cloud.debian.org/images/cloud/trixie-backports/latest/debian-13-backports-genericcloud-amd64.qcow2
```

Создадим машину

```bash
qm create 9000 --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci --name debian13cloud --serial0 socket --cpu cputype=x86-64-v3 --cores 4
```

```bash
qm disk import 401 debian-13-backports-genericcloud-amd64.qcow2 Samsung_SSD
qm set 401 --scsi0 Samsung_SSD:vm-401-disk-0,ssd=1
qm set 401 --ide2 Samsung_SSD:cloudinit
```
Настроим и запустим машину

```bash
qm set 401 --ciuser pavel --cipassword password --sshkeys /root/.ssh/authorized_keys --ipconfig0 ip=dhcp --ciupgrade 0
qm cloudinit update 401
```

```bash
qm start 401
qm terminal 401

# Логинимся, настроим qemu-guest-agent и кэш по вкусу

echo 'Acquire::HTTP::Proxy "[http://10.0.10.6:3142";'](http://10.0.10.6:3142";'/) >> /etc/apt/apt.conf.d/01proxy && echo 'Acquire::HTTPS::Proxy "false";' >> /etc/apt/apt.conf.d/01proxy
sudo -i
apt update
apt install qemu-guest-agent
poweroff

# Для выхода CTRL + O  
```

В дальнейшем можно выставить IP-адрес для нашей VM
```bash
qm set <VMid> --ipconfig<n> gw=<gateway ip>,ip=<device ip / CIDR>
qm cloudinit update <VMid>
```

Готово, можно создать template

```bash
qm template 401
```





