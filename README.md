
> [!WARNING]
> Все действия вы делаете на свой страх и риск. Автор не несет ответственности за поломку обороудования или другой потанцеальной проблеме.

## Зачем
Данный репозиторий нацелен на чуть более чем обычных пользователей с роутерами Keenetic, не вижу смысла расписывать для OpenWRT коих меньше(вижу смысл, позже допишу для OpenWRT юзеров). В моем user листе находятся все необходимые сайты для обхода youtube,discord и google play.

## Что такое nfqws
*nfqws* - утилита для модификации TCP соединения на уровне пакетов, работает через обработчик очереди NFQUEUE и raw сокеты.
*tpws (Transparent Proxy WebSocket)* — это утилита для прозрачного проксирования и модификации трафика, ориентированная на работу с HTTP/WebSocket-трафиком.

## Почему используем nfqws
При использовании в режиме TPWS есть недостатки, заключающиеся в том, что в этом режиме не модифицируется протокол QUIC (HTTP/3), а также приложение YouTube на телевизорах и игровых консолях не работает.

NFQWS - имеет ряд преимуществ перед TPWS в виде большего числа параметров модификации сетевых пакетов, а также модификации трафика по протоколу QUIC.

## Перво-наперво
 Подготовка роутера: 
* Понадобится свободная usb флешка, подойдет на 4гб.

* Установите [Entware](https://help.keenetic.com/hc/ru/articles/360021214160-Установка-системы-пакетов-репозитория-Entware-на-USB-накопитель) и допольнительные компоненты на вашем роутере от Keenetic. А именно : OPKG(Netfilter), протокол IPv6 и ssh-сервер.

* Также отключите сторонние фильтры в разделе Интернет-фильтры.

Далее все действия будут выполняться в среде Entware. Для подключения используем ssh установленный раннее.

* логин - root, пароль по умолчанию - keenetic, порт - 222 или 22

```
ssh root@192.168.1.1 -p 222
```

## Установка на зависимостей и нужных пакетов
1. Установка необходимых зависимостей (поочередно пропишите данные команды)

```
opkg update
opkg install ca-certificates wget-ssl
opkg remove wget-nossl
```

2. Создадим отдельную директорию для нашего репозитория и пропишем его в библеотеку.

```
mkdir -p /opt/etc/opkg
echo "src/gz nfqws-keenetic https://aaaaasdqwe123.github.io/Youtube-Discord-on-Keenetic/all" > /opt/etc/opkg/nfqws-keenetic.conf
```

Репозиторий универсальный, поддерживаемые архитектуры: `mipsel`, `mips`, `aarch64`, `armv7`, `x86`, `x86_64`, `lexra`.

3. Установим пакет
```
opkg update
opkg install nfqws-keenetic
```

4. По желанию можете устновить web-интерфейс для удобства редактирования файлов.
P.S. веб-интерфейс будет располагаться по адресу http://адрес_роутера:90 (по дефолту: http://192.168.1.1:90)



Обновление
```
opkg update
opkg upgrade nfqws-keenetic
opkg upgrade nfqws-keenetic-web
```

Удаление 
```
opkg remove --autoremove nfqws-keenetic-web nfqws-keenetic
```

Посмотреть информацию об установленной версии
```
opkg info nfqws-keenetic
opkg info nfqws-keenetic-web
```

Запуск/остановка/перезапуск/перезагрузка/просмотр статуса работы сервиса
```
service nfqws-keenetic {start|stop|restart|reload|status}
```
## Настройка
Конфиг файл находится по пути /opt/etc/nfqws/nfqws.conf . Можете установить `nano` или `vi` для редактирования.

```
# Provider network interface, e.g. eth3
# You can specify multiple interfaces separated by space, e.g. ISP_INTERFACE="eth3 nwg1"
ISP_INTERFACE="ppp0"

# All arguments here: https://github.com/bol-van/zapret (search for `nfqws` on the page)
# HTTP(S) strategy
NFQWS_ARGS="--dpi-desync=fakedsplit --dpi-desync-split-pos=1 --dpi-desync-ttl=0 --dpi-desync-repeats=16 --dpi-desync-fooling=md5sig --dpi-desync-fake-tls-mod=padencap --dpi-desync-fake-tls=/opt/etc/nfqws/tls_clienthello.bin"

# QUIC strategy
NFQWS_ARGS_QUIC="--filter-udp=443 --dpi-desync=fake --dpi-desync-repeats=11 --dpi-desync-fake-quic=/opt/etc/nfqws/quic_initial.bin"

# UDP strategy (doesn't use lists from NFQWS_EXTRA_ARGS)
NFQWS_ARGS_UDP="--filter-udp=50000-50099 --filter-l7=discord,stun --dpi-desync=fake"

# auto - automatically detects blocked resources and adds them to the auto.list
NFQWS_EXTRA_ARGS="--hostlist=/opt/etc/nfqws/user.list --hostlist-auto=/opt/etc/nfqws/auto.list --hostlist-auto-debug=/opt/var/log/nfqws.log --hostlist-exclude=/opt/etc/nfqws/exclude.list"

# list - applies rules only to domains in the user.list
#NFQWS_EXTRA_ARGS="--hostlist=/opt/etc/nfqws/user.list"

# all  - applies rules to all traffic except domains from exclude.list
#NFQWS_EXTRA_ARGS="--hostlist-exclude=/opt/etc/nfqws/exclude.list"

# IPv6 support
IPV6_ENABLED=0

# TCP ports for iptables rules
TCP_PORTS=443

# UDP ports for iptables rules
UDP_PORTS=443,50000:50099

# Keenetic policy name
POLICY_NAME="nfqws"
# Policy mode (0 - include, 1 - exclude)
POLICY_EXCLUDE=0

# Syslog logging level (0 - silent, 1 - debug)
LOG_LEVEL=0

NFQUEUE_NUM=200
USER=nobody
CONFIG_VERSION=6
```

Стратегии действуют для доменов из user.list и auto.list, минуя exclude.list. Настройка NFQWS_EXTRA_ARGS определяет три режима:

* list: Только домены из user.list

* auto: user.list + автоопределение недоступных ресурсов (3 сбоя за 60 сек)

* all: Весь трафик, кроме exclude.list


В случае если вы не собираетесь , что то редактировать в конфигурационном файле , то открывайте user.list

```
nano /opt/etc/nfqws/user.list
```

Прописываем данные сайтов , которые нужны.
```
youtube.com
youtu.be
googleapis.com
googlevideo.com
i.ytimg.com
i9.ytimg.com
yt3.ggpht.com
yt3.googleusercontent.com
yt4.ggpht.com
yt4.googleusercontent.com
gvt1.com
gstatic.com
youtube-ui.l.google.com
ytimg.l.google.com
ytstatic.l.google.com
*.discord.com
*.discord.gg
*.discord.media
*.discordapp.com
*.discordapp.net
cdn.discordapp.com
dis.gd
discord-activities.com
discord-attachments-uploads-prd.storage.googleapis.com
discord.app
discord.co
discord.com
discord.design
discord.dev
discord.gg
discord.gift
discord.gifts
discord.media
discord.new
discord.store
discord.tools
discordactivities.com
discordapp.com
discordapp.net
discordcdn.com
discordmerch.com
discordpartygames.com
discordsays.com
discordstatus.com
gateway.discord.gg
images-ext-1.discordapp.net
media.discordapp.net
www.discord.app
www.discord.com
cloudflare-ech.com
play.google.com
```

> [!NOTE]  
> Действия Все действия выше вы можете производить из web-интерфейса если устанавливали его выше.

Перезапускаем утилиту и проверяем работу наших сервисов.

```
service nfqws-keenetic restart
```



OpenWRT soon.

