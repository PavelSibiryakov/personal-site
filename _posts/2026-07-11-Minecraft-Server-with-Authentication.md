---
title: Minecraft сервер с Neoforge и авторизацией в docker
date: 2026-07-11 8:00:00 +0700
categories: [Games, How-tos]
tags: [docker, minecraft, lxc]     # TAG names should always be lowercase
---

Как я реализовал авторизацию на нашем небольшом сервере с мод паком с neoforge
# Оговорочки

- Использование eclipse-temurin:25-jre как image в docker (сервер может не перезагрузится автоматически при падении), хотя можно было использовать itzg/minecraft-server.  
- Мне пришлось запустить 2 версии моей сборки, т.к. на nanolimbo нельзя поставить моды, из-за чего neoforge отказывается подключать клиента, если на серверах за прокси различны моды
- Ники для пиратских клиентов должны быть исключительно нижнего регистра, если хотите включить whitelist

# Подготовка  

## Что скачать и что будем использовать?
- **VeloAuth** - <https://modrinth.com/plugin/veloauth> - Плагин
- **Velocity** - <https://papermc.io/downloads/velocity> - Прокси
- **NeoVelocity** - <https://modrinth.com/mod/neovelocity> - Modern Forwarding Scheme для Neoforge
## Файловая структура, с чем работаем (перед этим установить neoforge)

```
|-- neoforge
|   |-- config
|	|	`-- neovelocity-common.toml
|   `-- mods
|       `-- neovelocity-1.2.6+1.21.1-1.21.5.jar
|-- neoforge-auth
|   |-- config
|	|	`-- neovelocity-common.toml
|   `-- mods
|       `-- neovelocity-1.2.6+1.21.1-1.21.5.jar
`-- velocity
	|-- forwarding.secret		# После запуска
	|-- velocity.toml 			# После запуска
    `-- plugins
	    |-- veloauth			# После запуска
		|	`-- config.yml		# После запуска
		`-- veloauth-1.3.1.jar
```
  
## Docker compose
``` yaml
#docker-compose.yaml
services:
  velocity:
    image: eclipse-temurin:25-jre
    container_name: mc-velocity
    stdin_open: true
    tty: true
    working_dir: /data
    volumes:
      - ./velocity:/data
    ports:
      - "25565:25565"
    command: java -Xms512M -Xmx512M -jar velocity.jar
    restart: unless-stopped
    networks:
      - mc-internal
    
  neoforge-auth:
    image: eclipse-temurin:25-jre
    container_name: mc-neoforge-auth
    stdin_open: true
    tty: true
    working_dir: /data
    volumes:
      - ./neoforge-auth:/data
    command: java -Xms2G -Xmx8G @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.233/unix_args.txt
    restart: unless-stopped
    networks:
      - mc-internal
    
  neoforge:
    image: eclipse-temurin:25-jre
    container_name: mc-neoforge
    stdin_open: true
    tty: true
    working_dir: /data
    volumes:
      - ./neoforge:/data
    command: java -Xms4G -Xmx8G @user_jvm_args.txt @libraries/net/neoforged/neoforge/21.1.233/unix_args.txt
    restart: unless-stopped
    networks:
      - mc-internal

networks:
  mc-internal:
    driver: bridge
```
# Установка
- Запускаем `docker compose up -d`
- Останавливаем `docker compose down`
## Прокси
### Настройка прокси
- **velocity/velocity.toml**
	- Поменять порт для контейнера, должен совпадать с прописанным в docker-compse. Прослушиваем на всех интерфейсах `bind = "0.0.0.0:25565`
	- Выставить `player-info-forwarding-mode = "modern"`
	- В `[servers]` добавить 2 нащих сервера и их порты, добавить их в `try = ["auth","neoforge"]`
		```
		auth = "neoforge-auth:25568"
		neoforge = "neoforge:25567"
		```
	- В `[forced-hosts]` удалить все, но `[forced-hosts]` оставить
### Настройка плагина
- **velocity/plugins/veloauth/config.yml**
	- `auth-server:`
		- `server-name: auth`
## Сервера
- Выставить для всех **server-port** как указали в `[servers]` в настройках прокси
- Желательно сделать `enforce-secure-profile=false` в **server.properties**
- Cкопировать `velocity/forwarding.secret` и вставить его значение в `neoforge/config/neovelocity-common.toml` и `neoforge-auth/config/neovelocity-common.toml`
- neoforge-auth
	- Изменить **server.properties**
		``` 
		difficulty=peaceful
		force-gamemode=true
		gamemode=adventure
		generate-structures=false
		generator-settings={"layers"\:[],"biome"\:"minecraft\:the_void"}
		max-tick-time=-1
		max-world-size=1
		online-mode=false
		pvp=false
		simulation-distance=2
		view-distance=2
		```
- На этом этапе все готово к запуску
## После запуска
### neoforge-auth 

Подключаемся к контейнеру `docker attach mc-neoforge-auth`
Чтобы выйти **CTRL + P**, **CTRL + Q**
Прописываем настройки для сервера и ставим начальную платформу
```
gamerule doDaylightCycle false
gamerule doWeatherCycle false
gamerule doMobSpawning false
gamerule randomTickSpeed 0
gamerule announceAdvancements false
gamerule spectatorsGenerateChunks false
fill 0 -63 0 -1 -63 -1 minecraft:stone
```
Сервер готов

# Управление
`docker attach mc-neoforge-auth` или `docker attach mc-neoforge`. Выход **CTRL + P**, **CTRL + Q**
`docker compose start\stop\restart` - Запуск, выключение, перезагрузка