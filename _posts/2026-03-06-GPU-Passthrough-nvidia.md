---
title: GPU Passthrough в LXC контейнер с docker для видеокарт Nvidia
date: 2026-03-06 8:00:00 +0700
categories: [Linux, How-tos]
tags: [docker, proxmox, linux, nvidia, lxc]     # TAG names should always be lowercase
---
Уcтановим зависимости
```bash
apt update && apt install build-essential software-properties-common make pve-headers-$(uname -r) sudo -y
```

Для начала скачиваем драйвера c сайта nvidia.com/drivers

Устанавливаем их на proxmox сервер.
```bash
chmod +x NVIDIA-Linux-x86_64-***.run
./NVIDIA-Linux-x86_64-***.run
```
Рекомендую создать пользователя nvidia-persistenced

```bash
useradd -r -s /usr/sbin/nologin -M nvidia-persistenced
```

Мне потребовалось создать сервис в systemd, чтобы у меня драйвер нормально запускался при запуске

```bash
cat<<EOF > /etc/systemd/system/nv-persistenced.service
[Unit]
Description=NVIDIA Persistence Daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user nvidia-persistenced
ExecStop=/usr/bin/nvidia-persistenced --terminate
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable nv-persistenced.service
systemctl start nv-persistenced.service
```

Далее создаем по гайду docker lxc


После создания и перезагрузки proxmox нода проверим какие устройства nvidia появились в системе

```bash
ls -la /dev/nvidia*
```
``
```
crw-rw-rw- 1 root root 195,   0 Feb 18 19:51 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Feb 18 19:51 /dev/nvidiactl
crw-rw-rw- 1 root root 509,   0 Feb 18 19:51 /dev/nvidia-uvm
crw-rw-rw- 1 root root 509,   1 Feb 18 19:51 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
drwxr-xr-x  2 root root     80 Feb 18 19:51 .
drwxr-xr-x 22 root root   5740 Feb 18 19:51 ..
cr--------  1 root root 234, 1 Feb 18 19:51 nvidia-cap1
cr--r--r--  1 root root 234, 2 Feb 18 19:51 nvidia-cap2
```

Нам нужно пропустить все устройства в lxc и дать больше чем 1024 мб памяти (Не сможет установщик разархивировать)

```bash
pct set 112 --memory 2048 --dev0 /dev/nvidia0 --dev1 /dev/nvidiactl --dev2 /dev/nvidia-uvm --dev3 /dev/nvidia-uvm-tools --dev4 /dev/nvidia-caps/nvidia-cap1 --dev5 /dev/nvidia-caps/nvidia-cap2
```


В proxmox ноде записываем драйвер в lxc
```bash
pct push 112 NVIDIA-Linux-x86_64-580.126.18.run /root/NVIDIA-Linux-x86_64-580.126.18.run
```
Подключаемся в lxc и установим драйвера
```bash
chmod +x /root/NVIDIA-Linux-x86_64-580.126.18.run
/root/NVIDIA-Linux-x86_64-580.126.18.run --no-kernel-modules
reboot
```
Проверим установку
```bash
nvidia-smi
```
Если все ок, то увидите такую картину. К сожалению она не будет показывать запущенные процессы в lxc, но покажет все, если запустить на proxmox ноде.
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.18             Driver Version: 580.126.18     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 1070        Off |   00000000:05:00.0 Off |                  N/A |
|  0%   51C    P0             32W /  151W |       0MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```
Теперь установим nvidia container toolkit (нужно только для использования nvidia видеокарты в docker)

```bash
apt install gnupg2 -y
```
Добавим репозиторий nvidia-container-toolkit
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt-get update
```
Установим nvidia-container-toolkit
```bash
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.18.2-1
  sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

создать в /etc/docker/daemon.json

```json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    },
    "storage-driver": "fuse-overlayfs"
}
```

Теперь при деплое сервиса, нужно просто в docker-compose.yml добавить

```yaml
deploy:
      resources:
        reservations:
          devices:
          - capabilities: [gpu]
```

Все, готово

P.S. Многие рекомендую установить nvidia-cuda-toolkit <https://developer.nvidia.com/cuda-downloads> для работы с cuda. Мне не потребовалось.