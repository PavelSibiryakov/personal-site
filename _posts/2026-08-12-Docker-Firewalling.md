---
title: Firewall с Docker на сервере
date: 2026-07-12 7:00:00 +0700
categories: [How-tos, Security, Linux]
tags: [docker, linux, security]     # TAG names should always be lowercase
---
В этом небольшом гайдике расскажу, как я защитил свои машины поверх Firewall на сети, с какими трудностями я столкнулся.
## Реализация
Идею взял отсюда: <https://github.com/chaifeng/ufw-docker>

Я решил пойти по пути наименьшего сопротивления: **ufw**
```shell
apt update && apt install ufw -y
ufw allow ssh # Желательно ограничить ssh до вашего vpn туннеля
ufw enable
```



К сожалению, **docker** самостоятельно переписывает (ip/nf)tables правила, поэтому нам нужно изменить конфигурацию **ufw**.
```shell
cp /etc/ufw/after.rules /etc/ufw/after.rules.bak # backup старых правил

cat > /etc/ufw/after.rules<<EOF
#
# rules.input-after
#
# Rules that should be run after the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-after-input
#   ufw-after-output
#   ufw-after-forward
#

# Don't delete these required lines, otherwise there will be errors
*filter
:ufw-after-input - [0:0]
:ufw-after-output - [0:0]
:ufw-after-forward - [0:0]
:DOCKER-USER - [0:0]
:ufw-user-forward - [0:0]
# End required lines

# don't log noisy services by default
-A ufw-after-input -p udp --dport 137 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp --dport 138 -j ufw-skip-to-policy-input
-A ufw-after-input -p tcp --dport 139 -j ufw-skip-to-policy-input
-A ufw-after-input -p tcp --dport 445 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp --dport 67 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp --dport 68 -j ufw-skip-to-policy-input

# don't log noisy broadcast
-A ufw-after-input -m addrtype --dst-type BROADCAST -j ufw-skip-to-policy-input

-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A DOCKER-USER -m conntrack --ctstate INVALID -j DROP
-A DOCKER-USER -i eth0 -j DROP

# don't delete the 'COMMIT' line or these rules won't be processed
COMMIT
EOF
```
Теперь чтобы разрешить доступ к нашему контейнеру просто прописываем:

**Обратите внимание, что указывается внутренний порт контейнера**, изменить его у меня не получилось.

```shell
ufw route allow from 192.168.x.x to any port 2283 proto tcp
ufw route allow from 192.168.x.x to any port 443
ufw route allow 80/tcp
``` 
Применяем изменения
```shell
ufw reload
```
На этом гайд заканчивается, дальше расскажу с чем мне пришлось столкнуться.

## Боль, с которой я столкнулся
**Всё, что тут находится не рекомендуется к использованию**
### Проблемы с ufw
Изначально я решил выбрать **ufw**, из-за его простого синтаксиса, но сразу после написания всех правил, оказалось, что docker'у вообще по барабану на наши правила, он слушал на всем интерфейсе. Даже тогда, когда я указал его подсеть в docker-compose.yaml секции ports, ничего не менялось. 
А самое печальное, что я это заметил только спустя несколько месяцев после установки, когда мне рассказал мой хороший товарищ о своей реализации firewall на голом nftables. (( Рома, привет! )) Ведь часть сервисов реально блокировалось: samba, ssh, apt-cacher-ng, т.к. они были запущены просто на lxc без docker.
### Костыль в виде firewalld
Затем я увидел на <https://docs.docker.com>, что docker имеет интеграцию с firewalld. Естественно я попробовал реализовать firewall на нем. Docker создал отдельную zone и policy. Я поставил глобальную zone на drop, создал зону для ssh и прочего (для тестирования), добавил все подсети и порты, нужные мне. Но docker не подчинялся правилам этой зоны, поэтому я попробовал изменить его zone и policy. Я запретил forward из any, сделал rich-rule, который позволял доступ только одному хосту, но ничего не помогло. Тогда я его и снёс
### Отключение изменение iptables docker'ом
Это реально помогло на многих моих машинах, но вот несколько контейнеров отказались работать в таком режиме, из-за чего мне пришлось искать замену этому способу.
### Возвращение к ufw и проблемы с iptables
Я пришел к этой конфигурации
```shell
sudo iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -I DOCKER-USER 2 -p tcp --dport 2283 -s 192.168.x.x -j ACCEPT
sudo iptables -A DOCKER-USER -i eth0 -j DROP
```
С последней строчкой у меня было много казусов: в iptables важен порядок правил, но я почему-то решил разместить ее первой -- сломал весь docker.
При этом chatGPT постоянно предлагал этот вариант, но без `-i eth0`, что ломало сервисы, состоящих из нескольких контейнеров. 
Дальше я обнаружил на забытом <https://stackoverflow.com> ответ от 2018 года, а затем и нашел описанный в моем гайде способ, который я модифицировал под свои нужды
## Небольшой комментарий
iptables под своим капотом на современном ядре Linux уже имеет nftables, поэтому правила писал под них
