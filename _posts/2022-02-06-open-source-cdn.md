---
layout: post
title: Опыт реализации Open Source OTT TV CDN решения из 2015 года
tags: open source ott tv cdn
author: github:freefd, DmitryZaytsev
---

В основе [CDN](https://ru.wikipedia.org/wiki/Content_Delivery_Network) <sup id="a1">[1](#f1)</sup> лежит принцип географически более близкого расположения серверов с контентом к пользователю, что увеличивает скорость доставки контента и снижает нагрузку на первичный источник (Origin).

> *Внимание:*
* Статья является преработанной внутренней документации in-house [OTT](https://ru.wikipedia.org/wiki/OTT) <sup id="a2">[2](#f2)</sup> TV CDN решения из 2015 года от российского [ISP](https://ru.wikipedia.org/wiki/Интернет-провайдер) <sup id="a3">[3](#f3)</sup> федерального масштаба для Live трафика OTT.
* На 2022 год решение уже притерпело изменения, которые глобально не сказались на основополагающих архитектурных аспектах.
* Статья не описывает состав и принцип работы OTT [Middleware](https://ru.wikipedia.org/wiki/Связующее_программное_обеспечение) <sup id="a4">[4](#f4)</sup>.
* Решение работает в пределах одной [ASN](https://ru.wikipedia.org/wiki/Автономная_система_(Интернет)) <sup id="a5">[5](#f5)</sup>.<br/>
* Реальное название бренда заменено на [ACME](https://ru.wikipedia.org/wiki/Acme_Corporation) <sup id="a6">[6](#f6)</sup>.


Упрощённое объяснение разницы между классическим распространением контента с единичного сервера:

![NOCDN](/images/2022-02-06-open-source-cdn-1.png)

и через CDN:

![CDN](/images/2022-02-06-open-source-cdn-2.png)


## Составляющие CDN для ACME
### Функциональные
В иерархии CDN реализации для ACME можно выделить следующие функциональные составляющие:

#### Origin Streamer
Физически представляет собой сервер с большим количеством 7200 [RPM](https://ru.wikipedia.org/wiki/Оборот_в_минуту) <sup id="a7">[7](#f7)</sup> дисков для хранения и раздачи контента. Контент на стримеры доставляется по [NFS](https://ru.wikipedia.org/wiki/Network_File_System) <sup id="a8">[8](#f8)</sup> 10G с [Packager'ов](https://ottverse.com/abr-packaging-for-vod-live-hls-and-dash-overview/#Streaming_Packagers) <sup id="a9">[9 (EN)](#f9)</sup>. То есть на серверах находится prepackaged контент, уже разбитый на сегменты (chunks). Раздаёт media [m3u8](https://ru.wikipedia.org/wiki/M3U) <sup id="a10">[10](#f10)</sup> манифест и контент для конечного пользователя.

> *Внимание:* <br/>
> На 2022 год это не самый лучший вариант стриминга. Например, можно использовать пакетированиее на лету с помощью [NGINX-based VOD Packager](https://github.com/kaltura/nginx-vod-module) <sup id="a11">[11 (EN)](#f11)</sup>

#### CDN Precache
Выполняет две функции:
* Ограничивает трафик в сторону Origin Streamer'ов, защищая от перегрузки канала и превышения лицензионных ограничений.
* Обеспечивает отказоустойчивость и балансировку между Origin Streamer'ами.
Представляет собой виртуальную машину или физический сервер с большим количеством [RAM](https://ru.wikipedia.org/wiki/Запоминающее_устройство_с_произвольным_доступом) <sup id="a12">[12](#f12)</sup>, куда кешируется контент в момент проксирования пользовательских запросов, и 10G+ Ethernet.

#### CDN Cache
Горизонтально масштабируемая единица комплекса, непосредственно раздающая контент пользователям. Представляет собой виртуальный или физический сервер с умеренным количеством RAM, куда кешируется контент в момент проксирования пользовательских запросов, и 1G или 10G+ Ethernet, в зависимости от технических возможностей сервера.

#### Router
Каналообразующая единица комплекса, принимающая анонсы [BGP Anycast](https://ru.wikipedia.org/wiki/Anycast) <sup id="a13">[13](#f13)</sup> от ближайших CDN Cache'ров, маршрутизирующая и балансирующая пользовательский трафик до них. Также организовывает отказоустойчивость решения, выбирая для пользовательского трафика новый путь до CDN Cache'ров в случае выхода из строя ближайших.

### Программные
Вся программная реализация CDN использует Open Source продукты:
#### Операционная Система
В качестве платформы все серверы, за исключением Origin Streamers, используют [CentOS](https://centos.org/) <sup id="a14">[14 (EN)](#f14)</sup> - дистрибутив Linux, основанный на коммерческом Red Hat Enterprise Linux компании Red Hat и совместимый с ним. Origin Streamer'ы используют [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) <sup id="a15">[15 (EN)](#f15)</sup>.
> *Внимание*: <br/>
> В результате принятых Red Hat решений, c 2021 года CentOS 8 трансформирован в [CentOS Stream](https://www.centos.org/centos-stream/) <sup id="a16">[16 (EN)](#f16)</sup> и считается платформой для разработки изменений, включаемых затем в RHEL редакцию. Более не рекомендуется как промышленная полноценная замена RHEL.

#### Кеширующий HTTP-прокси сервер
Доставку OTT-контента и кеширование в процессе проксирования выполняет [Nginx](http://nginx.org/ru/) <sup id="a17">[17](#f17)</sup> - HTTP-сервер и обратный прокси-сервер, почтовый прокси-сервер, а также TCP/UDP прокси-сервер общего назначения. Серверы Nginx присутствуют на Origin Streamer'ах, CDN Precache'рах и CDN Cache'рах.

#### Демон динамической IPv4/IPv6 маршрутизации
Для задач динамической маршрутизации IPv4/IPv6 на платформе Linux в CDN для ACME используется [The BIRD Internet Routing Daemon](http://bird.network.cz/) <sup id="a18">[18 (EN)](#f18)</sup>. С его помощью CDN Cache'ры анонсируют BGP Anycast IP.
> *Внимание*: <br/>
> В 2022 году, пожалуйста, используйте BIRD2 реализацию.

#### Система мониторинга daemon'ов
[Monit](https://mmonit.com/monit/) <sup id="a19">[19 (EN)](#f19)</sup> выполняет локальных мониторинг сервисов, позволяет выполнять действия при достижении определенных событий. Отслеживает работоспособность Nginx.

#### Система мониторинга
[Zabbix](http://www.zabbix.com/ru/) <sup id="a20">[20](#f20)</sup> отслеживает статусы сервисов ОС серверов и ключевые метрики комплекса. Считает статистику по трафику для Live/[VoD](https://ru.wikipedia.org/wiki/Видео_по_запросу) <sup id="a21">[21](#f21)</sup>/PVR, [Cache Hit Ratio](https://www.cloudflare.com/learning/cdn/what-is-a-cache-hit-ratio/) <sup id="a22">[22 (EN)](#f22)</sup>, утилизацию кеша серверов, трафик из/в Origin.

> *Внимание*: <br/>
> В 2022 году имеет смысл рассмотреть альтернативы Zabbix, например, [Prometheus](https://prometheus.io/) <sup id="a23">[23 (EN)](#f23)</sup>.

## Архитектурная схема CDN для ACME

![ACME CDN Architecture](/images/2022-02-06-open-source-cdn-3.png)

Как видно на схеме, коммуникации Cache <-> Precache выполняются в формате Active-Active.

## Диаграмма последовательности потоков трафика пользовательского устройства
Для доставки фрагментов видеопотока используется классическая схема для HLS/DASH с двумя m3u8 манифестами: master и media.

![HLS/Dash ABR](/images/2022-02-06-open-source-cdn-4.png)

Подробней про HLS/DASH и ABR можно прочесть в [What’s HTTP Live Streaming](https://medium.com/@shoheiyokoyama/whats-http-live-streaming-8821d299ac04) <sup id="a24">[24 (EN)](#f24)</sup>. Общую последовательность запросов можно представить как

![ACME CDN Traffic Flow](/images/2022-02-06-open-source-cdn-5.png)

Важно заметить, что master и media манифесты не всегда обязан предоставлять Origin, порой это может быть совершенно отдельный ресурс. В этом случае Origin от конечного пользователя полностью скрыт CDN.

## Настройки программного обеспечения комплекса

### Упрощённая сетевая схема примера подключения Cache сервера к Router

![ACME CDN Cache to Router connectivity](/images/2022-02-06-open-source-cdn-6.png)
### BGP: Juniper MX

Примеры настроек [BGP](https://ru.wikipedia.org/wiki/Border_Gateway_Protocol) <sup id="a25">[25](#f25)</sup>- и [BFD](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) <sup id="a26">[26 (EN)](#f26)</sup>-сессий для [Juniper MX](https://www.juniper.net/ru/ru/products/routers/mx-series.html) <sup id="a27">[27](#f27)</sup> в классическом **``show configuration``** формате:
```javascript
protocols {
    ...
    bgp {
        ...
        group tv-cdn {
            type external;
            description "-- OTT TV CDN Servers --";
            mtu-discovery;
            log-updown;
            import import-tv-cdn;
            family inet {
                unicast {
                    prefix-limit {
                        maximum 5;
                        teardown idle-timeout 5;
                    }
                }
            }
            export export-default;
            multipath;
            neighbor 198.51.100.178 {
                description "-- msk-cache-01.tv.acme.tld";
                peer-as 64511;
                bfd-liveness-detection {
                    minimum-interval 250;
                }
            }
            ...
        }
        ...
    }
}
policy-options {
    prefix-list default {
        0.0.0.0/0;
    }
    policy-statement export-default {
        term default-route {
            from {
                prefix-list default;
            }
            then accept;
        }
        then reject;
    }
    policy-statement import-tv-cdn {
        term reject-default-route {
            from {
                route-filter 0.0.0.0/0 exact;
            }
            then reject;
        }
        term reject-multicast {
            from {
                route-filter 224.0.0.0/4 orlonger;
            }
            then reject;
        }
        term bgp {
            from protocol bgp;
            then {
                community set type-aggregatable-customer;
                community add acme-msk;
                accept;
            }
        }
    }
}
```

Опция [BGP Multipath](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/bgp-multipath.html) <sup id="a28">[28 (EN)](#f28)</sup> используется для отказоустойчивости и балансировки пользовательского трафика между несколькими серверами в локации. BFD-сессия настроена для более быстрого разрыва BGP-сессии и перестройки таблицы маршрутов в случае недоступности Cache сервера.

### BGP: CDN Cache
Пример настроек BIRD для CDN Cache'ров:
```javascript
protocol device {
    scan time 10;
}

protocol direct {
    interface "lo*";
}

protocol bgp msk_cache_01_tv_pop01_public_bgp {
    description "-- msk-r1-pop01.acme.tld irb.1010";
    router id 198.51.100.178;
    local as 64511;
    neighbor 198.51.100.177 as 64496;
    export where source = RTS_DEVICE;
    import all;
    source address 198.51.100.178;
    next hop self;
}

protocol bfd msk_r1_pop01_bfd {
        interface "eth0" {
                interval 250 ms;
                multiplier 3;
        };
        neighbor 198.51.100.177;
}

log syslog { debug, trace, info, remote, warning, error, auth, fatal, bug };
```

BIRD каждые 10 секунд будет перечитывать список интерфейсов и выбирать из них лишь **``lo``** интерфейсы, на которых как secondary добавлен Anycast IP адрес 198.51.100.190. Анонсируются в BGP лишь Directly Connected интерфейсы (**``RTS_DEVICE``**), которыми и являются **``lo``**. Импортируются в виртуальную таблицу BGP любые префиксы от пира.

BFD работает на интерфейсе **``eth0``**, настройки **``interval``** и **``multiplier``** идентичны настройкам Juniper MX.

### ОС: CDN Cache, CDN Precache
Как известно, на подключениях 1G используется кодирование [8b/10b](https://en.wikipedia.org/wiki/8b/10b_encoding) <sup id="a29">[29 (EN)](#f29)</sup>, то есть, на 10 байт переданной информации приходится 8 байт данных. Накладные расходы в объёмах переданного трафика в единицу времени для 1G интерфейса могут достигать 20%. Более предпочтительно использовать 10G+ интерфейсы, в которых используется уже [64b/66b](https://en.wikipedia.org/wiki/64b/66b_encoding) <sup id="a30">[30 (EN)](#f30)</sup> кодирование и накладные расходы не превышают 3-4%.

В зависимости от реализации, средняя продолжительность TS-фрагмента потока в HLS/DASH может быть от [2 до 15 секунд](https://bitmovin.com/mpeg-dash-hls-segment-length/) <sup id="a31">[31 (EN)](#f31)</sup>. В реализации используемой ACME платформы OTT Middleware продолжительность одного фрагмента составляет 8 секунд, а размер TS-фрагмента не превышает 9 MB, в зависимости от выбранного клиентом профиля. Средний размер фрагмента около 2.2 MB. 

Учитывая размер фрагмента, имеет смысл включать [Jumbo Frame](https://ru.wikipedia.org/wiki/Jumbo-кадр) <sup id="a32">[32](#f32)</sup>, но только на тех серверах, которые коммутируются с сетевым оборудованием с включенным Jumbo Frame.

При подключении сервера к каналообразующему оборудованию через интерфейс 1G необходимо обратить внимание на возможность работы оборудования с Jumbo Frame. Например, некоторые SFP-трансиверы [1000BASE-T](https://ru.wikipedia.org/wiki/Gigabit_Ethernet) <sup id="a33">[33](#f33)</sup> (они же **``SFP-T``**), имеют Hardware MTU в 1500 байт и любые попытки задействовать Jumbo Frame на стороне сервера не дадут результатов.

Типовые настройки сетевого интерфейса для CentOS 7:
```ini
$ cat /etc/sysconfig/network-scripts/ifcfg-eth0 
TYPE="Ethernet"
BOOTPROTO="static"
DEFROUTE="yes"
PEERDNS="no"
PEERROUTES="no"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="no"
IPV6_DEFROUTE="no"
IPV6_PEERDNS="no"
IPV6_PEERROUTES="no"
IPV6_FAILURE_FATAL="no"
NAME="eth0"
UUID="55084b5b-3a68-4b2d-89c3-833cbf1f73fb"
DEVICE="eth0"
DNS1=198.51.100.6
DNS2=203.0.113.6
DOMAIN="tv.acme.tld"
SEARCH="tv.acme.tld"
ONBOOT="yes"
IPADDR=198.51.100.178
NETMASK=255.255.255.248
GATEWAY=198.51.100.177
MTU=9000
```

Дополнительно следует увеличить размер очереди передачи сетевого интерфейса [txqueuelen](https://www.linuxjournal.com/content/queueing-linux-network-stack) <sup id="a34">[34 (EN)](#f34)</sup> (Transmit Queue Length) до значений, больших значения по умолчанию. К сожалению, классический параметр **``ETHTOOL_OPTS``** не позволяет изменять этот параметр (на 2015 год), поэтому значение **``txqueuelen``** будет задаваться через udev при загрузке ОС:

```bash
$ cat /etc/udev/rules.d/50-custom-txqueuelen.rules 
KERNEL=="eth*", RUN+="/sbin/ip link set %k txqueuelen 10000"
```

Не менее важной является корректировка настроек ядра Linux через [sysctl](https://ru.wikipedia.org/wiki/Sysctl) <sup id="a35">[35](#f35)</sup>:
```ini
$ cat /etc/sysctl.conf
# Максимальное количество file-handlers, которое система сможет утилизировать
fs.file-max=100000

# Использовать полный диапазон портов
net.ipv4.ip_local_port_range = 1024 65535

# Выключить быструю утилизацию сокетов в состоянии TIME_WAIT.
net.ipv4.tcp_tw_recycle = 0

# Разрешить переиспользовать уже существующие сокеты в состоянии TIME_WAIT
net.ipv4.tcp_tw_reuse = 1

# По 16MB на сокет для передачи и приёма данных
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Максимальное число запоминаемых запросов на соединение, 
# для которых не было получено подтверждения от клиента
net.ipv4.tcp_max_syn_backlog = 4096

# Включает SYN Cookies, защита от SYN флуда
net.ipv4.tcp_syncookies = 1

# Максимальное количество "backlogged" сокетов
net.core.somaxconn = 2048

# Максимальное количество пакетов, взятое со всех интерфейсов за один цикл поллинга
net.core.netdev_budget = 600

# Отключить TCP Selective Acknowledgments (SACK) для высокопроизводительной сети
net.ipv4.tcp_sack = 0

# Изменить приёмные буферы для сокетов
net.ipv4.tcp_rmem = 16384 349520 16777216
```

Очень важно держать параметр **``net.ipv4.tcp_tw_recycle``** выключенным, иначе пользователи не смогут смотреть контент одновременно с нескольких устройств с одного IP.

Все логи Nginx шлёт по **``syslog``** на хост msk-cache-analytics01.acme.tld. Эта функциональность появилась в общедоступном Nginx появилась в версии 1.7.1.

На всех серверах групп CDN Precache и CDN Cache используется [tmpfs](https://ru.wikipedia.org/wiki/Tmpfs) <sup id="a36">[36](#f36)</sup> для размещения кешируемых Nginx файлов. Для CDN Cache'ров:
```java
$ grep tmpfs /etc/fstab
tmpfs    /var/cache/nginx/ram    tmpfs   defaults,nodev,nosuid,size=7G 0 0
```

Для CDN Precache'ров:
```java
$ grep tmpfs /etc/fstab 
tmpfs    /var/cache/nginx/ram    tmpfs   defaults,nodev,nosuid,size=14G 0 0
```

### Monit: CDN Cache, CDN Precache
Основой текущей реализации CDN ACME является Nginx и от стабильности его работы зависит качество услуги. Несмотря на то, что используется стабильная ветка Nginx, в продуктивных системах могут происходить коллизии. В качестве локального сервиса слежения за работоспособностью Nginx выбран Monit из-за своей компактности и нетребовательности к ресурсам системы.

```lua
$ sudo monit status
The Monit daemon 5.14 uptime: 4d 22h 9m 

Process 'nginx'
  status                            Running
  monitoring status                 Monitored
  pid                               991
  parent pid                        1
  uid                               0
  effective uid                     0
  gid                               0
  uptime                            4d 22h 9m 
  children                          5
  memory                            1.4 MB
  memory total                      24.4 MB
  memory percent                    0.0%
  memory percent total              0.3%
  cpu percent                       0.0%
  cpu percent total                 9.4%
  port response time                0.000s to [127.0.0.1]:80 type TCP/IP protocol DEFAULT
  data collected                    Tue, 03 May 2016 23:09:57

System 'msk-cache-01.tv.acme.tld'
  status                            Running
  monitoring status                 Monitored
  load average                      [0.36] [0.47] [0.49]
  cpu                               3.7%us 7.8%sy 0.0%wa
  memory usage                      269.9 MB [3.3%]
  swap usage                        0 B [0.0%]
  data collected                    Tue, 03 May 2016 23:09:57
```

Monit работает на **``localhost:2812``**, доступ к управлению разрешён с localhost:
```lua
$ cat /etc/monit.d/monit
set httpd port 2812 and
     use address localhost
     allow localhost
```

Параметры мониторинга Nginx включают:
* Отсылку сообщений о событиях в рассылку **``monit@acme.tld``**
* Перезапуск Nginx в случае отсутствия ответов по адресу **``127.0.0.1:80``**
* Перезапуск до 2 раз в случае загрузки vCPU более 40%, при отсутствии изменений после перезапусков посылается email
* Перезапуск до 5 раз в случае загрузки vCPU более 60%, при отсутствии изменений после перезапусков посылается email
* Если прошло более 10 перезапусков в 10 циклах, мониторинг Nginx отключается
```lua
$ cat /etc/monit.d/nginx
# nginx
check process nginx with pidfile /run/nginx.pid
  start program = "/usr/bin/systemctl start nginx.service"
  stop  program = "/usr/bin/systemctl stop nginx.service"
  noalert monit@acme.tld
  if failed host 127.0.0.1 port 80 then restart
  if cpu is greater than 40% for 2 cycles then alert
  if cpu > 60% for 5 cycles then restart 
  if 10 restarts within 10 cycles then timeout
```

### logrotate: CDN Cache
Так как все Access лог-файлы шлются на удалённый сервер **``msk-cache-analytics01.acme.tld``**, а локально на CDN Cache'рах хранится только информация о кешировании, нет большого смысла хранить эти лог-файлы более двух дней:

```java
$ cat /etc/logrotate.d/nginx 
/var/log/nginx/*.log {
        daily
        missingok
        rotate 1
        compress
        delaycompress
        notifempty
        create 644 nginx adm
        sharedscripts
        postrotate
                [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
        endscript
}
```

### Nginx: CDN Cache
Точкой входа пользовательского трафика в CDN являются CDN Cache'ры. Настройки Nginx этих серверов выглядят следующим образом:
```java
$ cat /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    log_format cache '$upstream_cache_status';

    log_format CDN '{"ip": "$remote_addr", "host": "$host", "path": "$request_uri", "status": "$status", "user_agent": "$http_user_agent", "length": $bytes_sent, "date": "$time_iso8601"}';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    proxy_cache_path /var/cache/nginx/ram keys_zone=ram:10m inactive=1m max_size=6656m;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
            root   /usr/share/nginx/html;
            return 403;
        }
    }
}
```

Модификации подвергся параметр **``worker_rlimit_nofile``**, изменяющий ограничения на максимальное число открытых файлов для рабочих процессов. Дополнительно был создан новый формат лога **``CDN``**, формирующий лог в формате JSON. Создана кеш-область в RAM максимальным размером в 6.5 GB параметром **``proxy_cache_path``**. Данные в кеше удаляются каждую минуту при отсутствии обращений к ним.
Также стоит обратить внимание на формат логирования **``cache``**, который записывает в лог только содержимое переменной **``$upstream_cache_status``**. Этого значения достаточно для дальнейшего подсчёта cache hit ratio.

```java
$ cat /etc/nginx/conf.d/CDN.conf 
upstream cdn.tv.acme.tld {
    # msk-precache-01.tv.acme.tld
    server 198.51.100.162:80;

    # msk-precache-02.tv.acme.tld
    server 198.51.100.166:80;

    # msk-vs-01.tv.acme.tld
    server 192.0.2.56:80 backup;

    # msk-vs-02.tv.acme.tld
    server 192.0.2.57:80 down;
}

upstream pvr01-cdn.tv.acme.tld {
    # msk-precache-01.tv.acme.tld
    server 198.51.100.162:80;

    # msk-precache-02.tv.acme.tld
    server 198.51.100.166:80;
}

upstream pvr02-cdn.tv.acme.tld {
    # msk-precache-01.tv.acme.tld
    server 198.51.100.162:80;

    # msk-precache-02.tv.acme.tld
    server 198.51.100.166:80;
}

map $http_host $upstreamSet {
    hostnames;
    pvr01-cdn.tv.acme.tld    "pvr01-cdn.tv.acme.tld";
    pvr01-cdn.tv.acme.tld:80 "pvr01-cdn.tv.acme.tld";
    pvr02-cdn.tv.acme.tld    "pvr02-cdn.tv.acme.tld";
    pvr02-cdn.tv.acme.tld:80 "pvr02-cdn.tv.acme.tld";
    default "cdn.tv.acme.tld";
}

server {
    listen 80;
    server_name pvr01-cdn.tv.acme.tld pvr02-cdn.tv.acme.tld cdn.tv.acme.tld;

    # msk-cache-analytics01.acme.tld
    access_log syslog:server=203.0.113.221:514,facility=local7,tag=nginx,severity=info CDN;

    access_log /var/log/nginx/cache.log cache;
    error_log /var/log/nginx/error.log;

    location ~* \.(m3u8)$ {
        proxy_cache off;
        expires -1;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_pass http://$upstreamSet;
    }

    location / {
        index plst.m3u8;

        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }

        proxy_cache ram;
        proxy_cache_key $host$uri;
        proxy_cache_valid 5m;
        proxy_temp_path /var/cache/nginx/ram/temp;
        proxy_store_access user:rw group:rw all:r;
        proxy_method GET;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://$upstreamSet;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

В конфигурации виртуального хоста CDN Cache'ра для выбора upstream группы задействован механизм map, который подставляет в **``proxy_pass``** в переменную **``$upstreamSet``** требуемые значения. Виртуальный сервер обслуживает имена **``pvr01-cdn.tv.acme.tld``**, **``pvr02-cdn.tv.acme.tld``**, **``cdn.tv.acme.tld``**. Access лог-файлы шлются на удалённый хост **``msk-cache-analytics01.acme.tld``** по протоколу syslog. Файлы m3u8 не кешируются, остальные файлы кешируются на 5 минут. Ключом занесения файла в кеш является **``$host$uri``**, так как часть **``$query_string``** видоизменяется в процессе жизни элемента HLS/DASH потока. Дополнительно переписывается поле **``Connection``** и включается HTTP/1.1 для задействования механизма **``Keep-Alive``**, позволяя сократить количество соединений между пользовательским устройством и CDN Cache'ром, а также между CDN Cache'ром и CDN Precache'ром.

Особое внимание стоит обратить на блок map, в котором присутствуют описания хостов как с указанием порта, так и без. Это решение требуется для iOS (iPhone, iPad), которая по умолчанию в заголовок Host при запросе подставляет порт, даже если это 80 порт.

### Nginx: CDN Precache
Само появление CDN Precache'ров обязано лицензионным ограничениям платформы OTT Middleware по гигабитам трафика с Origin Streamer'ов. Nginx на CDN Precache'рах отрабатывает логику отказоустойчивости решения, выбора Origin Streamer'а и распределения трафика между ними по разного вида услугам:
* Live
* VoD и Catch-Up
* PVR

Основная конфигурация Nginx на CDN Precache'ре практически не отличается от таковой на CDN Cache'ре:
```java
$ cat /etc/nginx/nginx.conf 
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 2048;
}

http {
    log_format main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    log_format CDN '{"ip": "$remote_addr", "host": "$host", "path": "$request_uri", "status": "$status", "user_agent": "$http_user_agent", "length": $bytes_sent, "date": "$time_iso8601"}';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    proxy_cache_path /var/cache/nginx/ram keys_zone=ram:10m inactive=1m max_size=13824m;

    include /etc/nginx/conf.d/*.conf;
}
```
Отличия заключаются в размере кеша в **``13.5 GB``** и отсутствии default виртуального хоста. Основная конфигурация:
```java
$ cat /etc/nginx/conf.d/CDN.conf 
upstream cdn.tv.acme.tld {
    # msk-vs-01.tv.acme.tld
    server 192.0.2.56:80;

    # msk-vs-02.tv.acme.tld
    server 192.0.2.57:80 down;

    keepalive 1024;
}

upstream pvr01-cdn.tv.acme.tld {
    # msk-vs-01.tv.acme.tld
    server 192.0.2.56:80;
}

upstream pvr02-cdn.tv.acme.tld {
    # msk-vs-02.tv.acme.tld
    server 192.0.2.57:80;
}

map $http_host $upstreamSet {
    hostnames;
    pvr01-cdn.tv.acme.tld "pvr01-cdn.tv.acme.tld";
    pvr02-cdn.tv.acme.tld "pvr02-cdn.tv.acme.tld";
    default "cdn.tv.acme.tld";
}

server {
    listen 80 default_server;
    server_name _;

    access_log off;
    error_log /var/log/nginx/msk-precache-01.tv.acme.tld.error.log;

    location ~* \.(m3u8)$ {
        proxy_cache off;
        expires -1;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504 http_403 http_404;
        proxy_pass http://$upstreamSet;
    }

    location / {
        index plst.m3u8;

        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }

        proxy_cache ram;
        proxy_cache_key $host$uri;
        proxy_cache_valid 5m;
        proxy_temp_path /var/cache/nginx/ram/temp;
        proxy_store_access user:rw group:rw all:r;
        proxy_method GET;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504 http_403 http_404;
        proxy_pass http://$upstreamSet;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

Применяются все те же приёмы, которые использовались для CDN Cache'ров, но теперь upstream группы содержат лишь хосты Origin Streamer'ов.

## PVR и виртуальные кластеры OTT Middleware
Для разделения [PVR](https://ru.wikipedia.org/wiki/Цифровой_видеорекордер) <sup id="a37">[37](#f37)</sup> каналов по отдельным стримерам и более гибкого масштабирования услуг реализованы виртуальные кластеры:
* pvr01-cdn.tv.acme.tld
* pvr02-cdn.tv.acme.tld

**``pvr01-cdn.tv.acme.tld``** привязан к **``msk-vs-01.tv.acme.tld``** Origin Streamer'у, а **``pvr02-cdn.tv.acme.tld``** - к **``msk-vs-02.tv.acme.tld``**. Это позволяет разнести пользовательскую нагрузку между Origin Streamer'ами по услуге PVR, так как услуга PVR вызывает большую IO на дисковую подсистему Origin Streamer'ов, и, при необходимости, осуществить резервирование услуг по отдельности.

При запросе PVR контента Service Gateway **``sgw.tv.acme.tld``** в составе OTT Middleware (**``msk-sgw-01.tv.acme.tld``** и **``msk-sgw-02.tv.acme.tld``**) отдаёт абоненту в плейлисте ссылку **``pvr01-cdn.tv.acme.tld``** или **``pvr02-cdn.tv.acme.tld``**, в зависимости от того, на каком Origin Streamer'е находится контент. Абонент обращается на общий адрес CDN **``cdn.tv.acme.tld``** и попадает на CDN Cache'ры, далее запрос проксируется на CDN Precache'ры. CDN Precache'ры согласно настроенным правилам перенаправляют запрос по доменному имени либо на **``msk-vs-01.tv.acme.tld``**, либо на **``msk-vs-02.tv.acme.tld``**.

## Мониторинг
> *Внимание*: <br/>
> В 2022 году имеет смысл рассмотреть альтернативы Zabbix, например, Prometheus.
> Все изображения в тексте соответствуют таковым для 2016 года.

Для мониторинга характеристик CDN используется Zabbix с агентом на хостах. С каждого CDN Cache'ра и CDN Precache'ра помимо стандартных метрик, дополнительно снимаются метрики утилизации кеша Nginx, общее количество дискового пространства кеша Nginx, эффективность кеширования.

Создан хост **``cdn.tv.acme.tld``** с суммирующими данные с CDN Cache'ров/CDN Precache'ров элементами данных:
![ACME CDN Summary Host](/images/2022-02-06-open-source-cdn-7.png)

### CDN
Из суммируемого элемента данных по входящему трафику:

![ACME CDN Total Network Incoming](/images/2022-02-06-open-source-cdn-8.png)

и суммируемого элемента данных по исходящему трафику:

![ACME CDN Total Network Outgoing](/images/2022-02-06-open-source-cdn-9.png)

строится основной график потребления трафика в CDN:

![ACME CDN Total Networking](/images/2022-02-06-open-source-cdn-10.png)

Помимо общего графика для CDN, созданы общие региональные графики CDN. По ним можно оценивать количество трафика с региона или наличие коллизий с предоставлением сервиса в регионе.

Существует и общий суммирующий график утилизации Cache в CDN:

![ACME CDN Cache Free](/images/2022-02-06-open-source-cdn-11.png)

### Cache Hit Ratio

Для всех CDN Cache'ров снимаются значения Cache Hit Ratio в процентном отношении:
```ini
$ cat /etc/zabbix/zabbix_agentd.d/userparams.conf 
UserParameter=cache.ratio[*],echo "scale=2; $(/bin/grep HIT /var/log/nginx/cache.log | /usr/bin/wc -l)*100/$(/bin/grep -v -- - /var/log/nginx/cache.log | /usr/bin/wc -l)" | /usr/bin/bc
```

По снятым значениям для каждого CDN Cache'ра строится график:

![ACME CDN Cache Hit Ratio](/images/2022-02-06-open-source-cdn-12.png)

### Origin
Не менее важным является график трафика из/в Origin Streamer'ы, так как лицензирование платформы определяется количеством гигабит, потребляемых с Origin Streamer'ов:

![ACME CDN Origin Total Traffic](/images/2022-02-06-open-source-cdn-13.png)

График строится идентичным образом из двух суммирующих элементов данных: элемента данных входящего трафика с CDN Precache'ров:

![ACME CDN Origin Network Incoming](/images/2022-02-06-open-source-cdn-14.png)

и элементов данных исходящего трафика с CDN Precache'ров:

![ACME CDN Origin Network Outgoing](/images/2022-02-06-open-source-cdn-15.png)

Помимо этого, с каждого Origin Streamer'а снимаются значения по объёмам трафика услуг Live/PVR/VoD:

![ACME CDN Origin Services Traffic](/images/2022-02-06-open-source-cdn-16.png)

## Диагностика
> *Внимание:* <br/>
> В 2022 году для хранения логов имеет смысл отказаться от Syslog в пользу более гибких инструментов.
> Например, связка Nginx JSON logs + Fluentd + Apache Kafka + ClickHouse позволяет получить обработку логов в режиме реального времени и эффективное по месту хранение.

Для облегчения диагностики работоспособности CDN логи с CDN Cache'ров шлются на хост **``msk-cache-analytics01.acme.tld``** по протоколу **``syslog``**.

В качестве примера простейшей диагностики можно рассмотреть поиск по IP клиента CDN Cache'ра, который с 21 до 22 часов вечера приземлял трафик пользователя услуги PVR на платформе iOS:
```bash
$ grep 198.18.18.140 /var/log/remote/2016/05/15/*_21_access.log | grep -i pvr | grep -i apple | head -5
/var/log/remote/2016/05/15/msk-cache-03.tv.acme.tld_21_access.log:{"ip": "198.18.18.140", "host": "pvr01-cdn.tv.acme.tld", "path": "/pvr/id119_ACME_SG--tnt/02/c_1162603100204372.ts", "status": "200", "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)", "length": 1350008, "date": "2016-05-15T21:02:34+03:00"}
/var/log/remote/2016/05/15/msk-cache-03.tv.acme.tld_21_access.log:{"ip": "198.18.18.140", "host": "pvr01-cdn.tv.acme.tld", "path": "/pvr/id119_ACME_SG--tnt/02/c_1162603100204373.ts", "status": "200", "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)", "length": 1130888, "date": "2016-05-15T21:02:36+03:00"}
/var/log/remote/2016/05/15/msk-cache-03.tv.acme.tld_21_access.log:{"ip": "198.18.18.140", "host": "pvr01-cdn.tv.acme.tld", "path": "/pvr/id119_ACME_SG--tnt/01/plst.m3u8?chanId=119&startDate=15052016113500&endDate=15052016133500&curPos=15052016113500", "status": "200", "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)", "length": 35920, "date": "2016-05-15T21:02:37+03:00"}
/var/log/remote/2016/05/15/msk-cache-03.tv.acme.tld_21_access.log:{"ip": "198.18.18.140", "host": "pvr01-cdn.tv.acme.tld", "path": "/pvr/id119_ACME_SG--tnt/01/c_1162603100204004.ts", "status": "200", "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)", "length": 861207, "date": "2016-05-15T21:02:37+03:00"}
/var/log/remote/2016/05/15/msk-cache-03.tv.acme.tld_21_access.log:{"ip": "198.18.18.140", "host": "pvr01-cdn.tv.acme.tld", "path": "/pvr/id119_ACME_SG--tnt/02/plst.m3u8?chanId=119&startDate=15052016113500&endDate=15052016133500&curPos=15052016113500", "status": "200", "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)", "length": 35920, "date": "2016-05-15T21:02:38+03:00"}
```

Пользователь с IP **``198.18.18.140``** на устройстве **``Apple iPad``** с русскоязычной версией **``iOS 9.3.1``**, смотревший PVR канала ТНТ, приземлялся на сервере **``msk-cache-03.tv.acme.tld``**. Дополнительно можно сказать, учитывая информацию об имени PVR сервера, что пользователь уходил на Origin Streamer **``msk-vs-01.tv.acme.tld``**.

Для удобства чтения JSON-логов Nginx на сервере **``msk-cache-analytics01.acme.tld``** предустановлена утилита [jq](https://stedolan.github.io/jq/) <sup id="a38">[38 (EN)](#f38)</sup>.
```bash
$ grep -h 198.18.18.140 /var/log/remote/2016/05/15/*_21_access.log | grep -i pvr | grep -i apple | head -1 | jq .
{
  "ip": "198.18.18.140",
  "host": "pvr01-cdn.tv.acme.tld",
  "path": "/pvr/id119_ACME_SG--tnt/02/c_1162603100204372.ts",
  "status": "200",
  "user_agent": "AppleCoreMedia/1.0.0.13E238 (iPad; U; CPU OS 9_3_1 like Mac OS X; ru_ru)",
  "length": 1350008,
  "date": "2016-05-15T21:02:34+03:00"
}
```

## Ссылки
<b id="f1">1</b>. [Content Delivery Network](https://ru.wikipedia.org/wiki/Content_Delivery_Network) [↩](#a1)<br/>
<b id="f2">2</b>. [Over the Top](https://ru.wikipedia.org/wiki/OTT) [↩](#a2)<br/>
<b id="f3">3</b>. [Internet Service Provider](https://ru.wikipedia.org/wiki/Интернет-провайдер) [↩](#a3)<br/>
<b id="f4">4</b>. [Связующее программное обеспечение](https://ru.wikipedia.org/wiki/Связующее_программное_обеспечение) [↩](#a4)<br/>
<b id="f5">5</b>. [Автономная система](https://ru.wikipedia.org/wiki/Автономная_система_(Интернет)) [↩](#a5)<br/>
<b id="f6">6</b>. [Acme Corporation](https://ru.wikipedia.org/wiki/Acme_Corporation) [↩](#a6)<br/>
<b id="f7">7</b>. [Обороты в минуту](https://ru.wikipedia.org/wiki/Оборот_в_минуту) [↩](#a7)<br/>
<b id="f8">8</b>. [Network File System](https://ru.wikipedia.org/wiki/Network_File_System) [↩](#a8)<br/>
<b id="f9">9</b>. [Streaming Packagers
](https://ottverse.com/abr-packaging-for-vod-live-hls-and-dash-overview/#Streaming_Packagers) [EN] [↩](#a9)<br/>
<b id="f10">10</b>. [M3U формат](https://ru.wikipedia.org/wiki/M3U) [↩](#a10)<br/>
<b id="f11">11</b>. [NGINX-based VOD Packager](https://github.com/kaltura/nginx-vod-module) [EN] [↩](#a11)<br/>
<b id="f12">12</b>. [RAM](https://ru.wikipedia.org/wiki/Запоминающее_устройство_с_произвольным_доступом) [↩](#a11)<br/>
<b id="f13">13</b>. [BGP Anycast](https://ru.wikipedia.org/wiki/Anycast) [↩](#a12)<br/>
<b id="f14">14</b>. [CentOS](https://centos.org/) [EN] [↩](#a14)<br/>
<b id="f15">15</b>. [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) [EN] [↩](#a15)<br/>
<b id="f16">16</b>. [CentOS Stream](https://www.centos.org/centos-stream/) [EN] [↩](#a16)<br/>
<b id="f17">17</b>. [Nginx](http://nginx.org/ru/) [↩](#a17)<br/>
<b id="f18">18</b>. [The BIRD Internet Routing Daemon](http://bird.network.cz/) [EN] [↩](#a18)<br/>
<b id="f19">19</b>. [Monit](https://mmonit.com/monit/) [EN] [↩](#a19)<br/>
<b id="f20">20</b>. [Zabbix](http://www.zabbix.com/ru/) [↩](#a20)<br/>
<b id="f21">21</b>. [Видео по запросу](https://ru.wikipedia.org/wiki/Видео_по_запросу) [↩](#a21)<br/>
<b id="f22">22</b>. [Cache Hit Ratio](https://www.cloudflare.com/learning/cdn/what-is-a-cache-hit-ratio/) [EN] [↩](#a22)<br/>
<b id="f23">23</b>. [Prometheus](https://prometheus.io/) [EN] [↩](#a23)<br/>
<b id="f24">24</b>. [What’s HTTP Live Streaming](https://medium.com/@shoheiyokoyama/whats-http-live-streaming-8821d299ac04) [EN] [↩](#a24)<br/>
<b id="f25">25</b>. [Border Gateway Protocol](https://ru.wikipedia.org/wiki/Border_Gateway_Protocol) [↩](#a25)<br/>
<b id="f26">26</b>. [Bidirectional Forwarding Detection](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) [EN] [↩](#a26)<br/>
<b id="f27">27</b>. [Универсальные платформы маршрутизации серии MX](https://www.juniper.net/ru/ru/products/routers/mx-series.html) [↩](#a27)<br/>
<b id="f28">28</b>. [BGP Multipath](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/bgp-multipath.html) [EN] [↩](#a28)<br/>
<b id="f29">29</b>. [8b/10b Encoding](https://en.wikipedia.org/wiki/8b/10b_encoding) [EN] [↩](#a29)<br/>
<b id="f30">30</b>. [64b/66b Encoding](https://en.wikipedia.org/wiki/64b/66b_encoding) [EN] [↩](#a30)<br/>
<b id="f31">31</b>. [Optimal Adaptive Streaming Formats MPEG-DASH & HLS Segment Length
](https://bitmovin.com/mpeg-dash-hls-segment-length/) [EN] [↩](#a31)<br/>
<b id="f32">32</b>. [Jumbo-кадр](https://ru.wikipedia.org/wiki/Jumbo-кадр) [↩](#a32)<br/>
<b id="f33">33</b>. [Gigabit Ethernet](https://ru.wikipedia.org/wiki/Gigabit_Ethernet) [↩](#a33)<br/>
<b id="f34">34</b>. [Queueing in the Linux Network Stack](https://www.linuxjournal.com/content/queueing-linux-network-stack) [EN] [↩](#a34)<br/>
<b id="f35">35</b>. [Sysctl утилита](https://ru.wikipedia.org/wiki/Sysctl) [↩](#a35)<br/>
<b id="f36">36</b>. [Файловой хранилище tmpfs](https://ru.wikipedia.org/wiki/Tmpfs) [↩](#a36)<br/>
<b id="f37">37</b>. [Цифровой видеорекордер](https://ru.wikipedia.org/wiki/Цифровой_видеорекордер) [↩](#a37)<br/>
<b id="f38">38</b>. [Command-line JSON processor jq](https://stedolan.github.io/jq/) [EN] [↩](#a38)<br/>