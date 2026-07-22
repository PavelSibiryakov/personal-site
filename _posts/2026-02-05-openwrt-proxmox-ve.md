---
title: Openwrt как VM в Proxmox VE(CLI)
date: 2026-02-05 8:00:00 +0700
categories: [Linux, How-tos]
tags: [openwrt, linux, proxmox, vm]     # TAG names should always be lowercase
---
Скачиваем combined(ext4) образ с для x86/64
<https://firmware-selector.openwrt.org/>

Например
```bash
curl https://downloads.openwrt.org/snapshots//targets/x86/64/openwrt-x86-64-generic-ext4-combined.img.gz -o openwrt-x86-64-generic-ext4-combined.img.gz
```

Распаковать его
```bash
gunzip openwrt-x86-64-generic-ext4-combined.img.gz
```
```
gzip: openwrt-x86-64-generic-ext4-combined.img.gz: decompression OK, trailing garbage ignored
```

Найдем название нашего пула, запомним его
```
pvesm status
```
```
Name               Type     Status     Total (KiB)      Used (KiB) Available (KiB)        %
Samsung_SSD     zfspool     active       943476616       422794112       520682504   44.81%
local               dir     active        98497780        27410964        66037268   27.83%
local-lvm       lvmthin     active       335134720       249641852        85492867   74.49%
```

Создадим виртуальную машину, импортируем диск
```bash
qm create 400
qm disk import 400 openwrt-x86-64-generic-ext4-combined.img Samsung_SSD
```
Найдем как он называется в системе
```bash
qm config 400
```
```
boot:
meta: creation-qemu=10.1.2,ctime=1769329686
name: openwrt-test
smbios1: uuid=ccb826f8-dbe0-4232-a47a-6c4f1dfe810d
unused0: Samsung_SSD:vm-400-disk-0
vmgenid: 1fcf5591-bf72-4072-985e-789a7317c914
```

```bash
qm set 400 --name openwrt-test --scsi0 Samsung_SSD:vm-400-disk-0,ssd=1 --scsihw virtio-scsi-single --serial0 socket --cpu cputype=x86-64-v3
```
Увеличим размер диска до 1 гб (для OpenWrt достаточно)
```bash
qm disk resize 400 scsi0 1G
```
Определим наши интернет интерфейсы
```bash
ip -br -c a
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp4s0           UP
vmbr0            UP             fe80::b62e:99ff:feec:ce6f/64
vmbr0.100@vmbr0  UP             192.168.2.130/32 fe80::b62e:99ff:feec:ce6f/64
```
Добавим интерфейс к нашей машине, я поставлю vlan tag 10
```bash
qm set 400 --net0 model=e1000,bridge=vmbr0,tag=10	
```
Тут вместо model можно использовать virtio, rtl8139 (Для легаси), e1000e

Запускаем машину
```bash
qm start 400
```
Заходим в терминал
```bash
qm terminal 400
```

Зайдем в консоль openwrt нажатием enter после сообщения

Выставим новые настройки

```bash
vi /etc/config/network
```
Произведем соответсвующие изменения
```
config interface 'wan'
        option device 'eth0'
        option proto 'dhcp'

config device
        option name 'br-lan'
        option type 'bridge'
        list ports ''
```
Изменим настройки firewall

```bash
vi /etc/config/firewall
```
Обновим пакеты, установим luci и расширим ext4 раздел
```bash
apk update
apk add luci
apk add parted losetup resize2fs blkid
wget -U "" -O expand-root.sh "https://openwrt.org/_export/code/docs/guide-user/advanced/expand_root?codeblock=0"
. ./expand-root.sh
sh /etc/uci-defaults/70-rootpt-resize
```
Готово


