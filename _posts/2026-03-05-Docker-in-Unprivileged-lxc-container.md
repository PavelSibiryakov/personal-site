---
title: Docker в Unprivileged LXC контейнере
date: 2026-03-05 8:00:00 +0700
categories: [Linux, How-tos]
tags: [docker, proxmox, linux, lxc]     # TAG names should always be lowercase
---
обновим списки контейнеров и покажем доступные template

```bash
pveam update && pveam available
```
Я выберу этот
```
system          debian-13-standard_13.1-2_amd64.tar.zst
```
Скачаем в local 
```bash
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```
Найдем полное название
```bash
pveam list local
```

```
local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst
```

Создадим lxc контейнер
```bash
pct create 112 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst --cmode shell --cores 2 --hostname docker-test --memory 512 --unprivileged 1 --ostype debian --net0 name=eth0,bridge=vmbr0,gw=10.0.10.1,ip=10.0.10.11/24,firewall=1,tag=20 --ssh-public-keys /root/.ssh/authorized_keys --storage Samsung_SSD --features fuse=1,keyctl=1,nesting=1 --rootfs Samsung_SSD:10 --onboot 1
``` 
Запустим контейнер и зайдем в консоль

```bash
pct start 112 && pct console 112
```

Я добавлю свой кэш
```bash
echo 'Acquire::HTTP::Proxy "[http://10.0.10.6:3142";'](http://10.0.10.6:3142";'/) >> /etc/apt/apt.conf.d/01proxy && echo 'Acquire::HTTPS::Proxy "false";' >> /etc/apt/apt.conf.d/01proxy

apt update && apt upgrade
```

Начнем настраивать docker репозиторий

```bash
apt update
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: http://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
```

Установим docker
```bash
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin fuse-overlayfs
```
Поставим другой storage драйвер
```bash
cat <<EOF >/etc/docker/daemon.json 
{
  "storage-driver": "fuse-overlayfs"
}
EOF
systemctl restart docker.service
```
Проверим работу
```bash
docker run helloworld
```
Создадим другого пользователя
```bash
useradd -m -d /home/docker-user -U -s /bin/bash docker-user
usermod -aG docker docker-user
mkdir /docker
mkdir /home/docker-user/.ssh
cp ~/.ssh/authorized_keys /home/docker-user/.ssh/authorized_keys
chown -R docker-user:docker-user /home/docker-user/.ssh/
chown -R docker-user:docker /docker/
chmod 700 /home/docker-user/.ssh/
chmod 600 /home/docker-user/.ssh/authorized_keys
```
Готово

