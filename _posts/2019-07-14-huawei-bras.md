---
layout: post
title:  "Конфигурация оборудования Huawei BNG"
tags: BNG huawei edsg
author: "riddler63"
---

Disclaimer: Автор не раскрывает принципы работы BNG в целом и предполагает что читатель так или иначе представляет себе работу BNG и протокола RADIUS

Основные понятия и пример базовой настройки Huawei BNG (ME60/NE40/CX600) V6R9+

## Основные понятия

### Понятие группы пользователей (user-group)

User-group это ключевое слово, определенное на BNG используемое для идентификации сервиса присвоенного абоненту. User-group необходима для контроля прав доступа различных абонентов и используется при написании ACL (UCL).
User-group может быть добавлена в определенный домен или передана с RADIUS сервера в атрибуте Filter-ID.
User-group создается в system-view.

### EDSG (Enhanced dynamic service gateway)

Enhanced dynamic service gateway выполняет следующие функции на основе сервис групп (service-group) назначенных на абонентскую сессию пользователя.

Основные функции EDSG:

1. Accounting на основе адреса назначения и тарифного плана
2. Ограничение скорости на основе адреса назначения
3. Применение ограничения по времени для действия сервиса и задание его приоритета

EDSG обрабатывает трафик абонентской сессии на основе UCL где в качестве source и destination может быть задана service-group. До 8 таких сервисов может быть назначено на абонентскую сессию, но как правило хватает 3-4. Сервису так же можно задать time-range и делать accounting отдельно от мастер сессии, при этом в Accounting сообщениях будет указана мастер сессию.

Правила обработки трафика по service-group приоритетнее, чем по user-group и по умолчанию BNG не производит обработку трафика по user-group если на абонентскую сессию назначена service-group и трафик проматчился одним из EDSG сервисов. Данное поведение может быть изменено командой `traffic match user-group [ inbound | outbound ]`

Так же, по умолчанию ограничение по скорости EDSG сервиса добавляется к ограничению по скорости мастер сессии (M + S1 + S2 + S3 > M),однако это поведение может быть заменено командой `edsg traffic-mode rate` и EDSG сервис будет ограничивать трафик согласно конфигурации, однако общий трафик абонента не выйдет за пределы ограничения на мастер сессии. (M + S1 + S2 + S3 <= M)

#### Отличие от Cisco ISG

Классификация трафика и применение политик, сервисов осуществляется на основе user-group и service-group, а не по Source IP, поэтому при использовании разных VPN соблюдать не пересекающиеся адресные пространства на доступе не обязательно.

**Чтобы поменять сервис уже аутентифицированному пользователю достаточно через CoA поменять ему user-group и/или service-group или домен.**

### Понятие АCL

ACL-access control list используются для идентификации трафика. На BNG ACL используются в классификаторах трафика (traffic classifier) чтобы выделить определенный трафик.

#### Типы ACL

1. Интерфейсные ACL ( 1000 – 1999)
2. Базовые Basic (2000 – 2999)
3. Расширенные Adv (3000 – 3999)
4. Расширенные именованные Adv named (42768 to 75535)
5. Ethernet Ethernet frame header-based (4000 – 4999)
6. UCL (User-based ACL) на основе группы пользователей (6000-6999)
7. EDSG ACL (7000-7999)
8. MPLS ACL (10000 – 10999)

### Классификаторы трафика (traffic classifiers)

Классификатор трафика это перечень условий, которые должны быть выполнены, чтобы трафик был определен как принадлежащий данному классу.
Трафик может быть классифицирован по специфичным полям в заголовках фреймов, пакетов и пр.
Несколько условий попадания трафика в определенный класс может быть определено. По умолчанию, логический оператор ИЛИ используется для правил внутри одного классификатора.

### Действие (Behavior)

Одно или несколько действий применяемое к классифицируемому трафику. Какие-то действия могут присутствовать в одном Behavior, какие-то взаимоисключающие. Основные перечислены ниже:

1. Deny or permit (Запретить или разрешить)
2. CAR (Ограничить полосу, Policing)
3. http-redirect (Перенаправление HTTP запросов)
4. nat bind instance ( Перенаправить трафик в NAT)
5. Remark ( Перемаркировать поля отвечающие QoS в заголовках и не только DSCP, EF, 8021p, ip df и так далее)
6. Service class (Назначить внутренний сервисный класс, от 0 до 7)
7. Redirect to VPN ( Перенаправление в VPN)
8. IPv4/IPv6 strong redirection (PBR, перенаправление на IP, в интерфейс)
9. IPv4 multiply next hop redirection ( множественное перенаправление, на несколько возможных адресов)
10. Security ( URPF и пр)
11. Queue scheduling (HQoS)

### Трафиковые политики

Traffic policy это политика QoS сформированная путем ассоциации классификатора трафика (classifier) и действия (behavior) применяемого к этому трафику. Traffic policy может быть применена на интерфейс, на устройство глобально или к отдельному абонентскому сервису.
BNG поддерживает динамическое изменение правил политики трафика, но не поддерживает динамического изменения типа политики Shared/Non-shared. При изменении ACL задействованных в политике новые правила вступают в действия сразу.

### Group Identifier

Group Identifier (GID - Идентификатор группы) это сопоставление индекса user-group назначенного абоненту в data-plane в PFE (Packet Forward Engine). Этот GID является триггером для включения процесса проверки трафика по ACL. Если абонент имеет группу абонентов, то трафик этого пользователя будет проверен ACL (UCL).

![GID](/images/GID2.png)

Ключевых полей обычно 5, это SRC_IP, DST_IP, SRC_PORT, DST_PORT и PROTOCOL_TYPE. BNG реализует функционал ACL в TCAM (Ternary Content-addressable memory). Содержимое TCAM имеет три части, первая это правило из политики загруженное в PFE(Packet Forwarding Engine), второе это маска данного правила, маскирующая соответствующие биты, а последняя часть формируется из параметром исследуемого пакета.

В случае если в трафиковой политике определено более одного правила сочетающего классификатор трафика и действие, использование ключевого слова precendence позволяет контролировать какие правила в каком порядке будут выполняться. Меньшее значение precedence говорит о более высоком приоритете.

### Service Group Identifier (EDSG)

Service Group Identifier похож на User-group identifier, однако precedence в трафиковых политиках не имеет значение при указании Priority при аутентификации Service на Radius сервере.

### Shaping(user-queue) vs Policing (car)

По умолчанию Huawei BNG делает shaping в downstream (что хорошо для TCP) и policing (что нормально) в upstream направлении.
Поведение по умолчанию можно поменять как для user-group `rate-limit-mode { car | user-queue } outbound`, так и для service-group `service rate-limit-mode { car | user-queue } { inbound | outbound }`.

Если на BNG установлен FPIC c eTM чипом, то downstream traffic shaping и другие операции с абонентским трафиком обычно выполняемые на TM чипе основной платы (LPU) выполняются на eTM чипе FPIC. Данное поведение можно отключить (но зачем?)

### Домен

Домен является агрегирующей сущностью, шаблоном со значениями некоторых атрибутов по умолчанию. В домене так же определяются некоторые параметры NAT, порядок обработки правил на основе user-group/service-group, а так же тип (policing/shaping) ограничения полосы пропускания в разных направлениях (upstream/downstream)

Каждый домен может содержать следующие параметры (далеко не полный список)

1. Имя IPv4 pool/IPv4 pool group
2. Имя IPv6 Pool
3. Параметры DNSv6
4. Имя NAT instance для правильного RADIUS accounting
5. Имя схемы аутентификации
6. Имя схемы аккаунтинга
7. Имя Radius server group
8. Имя user-group
9. Shaping vs policing для user-group и service-group в upstream/downstream направлении
10. Метод подсчёта трафика для IPv4 и IPv6
11. Метод распределения bandwidth между основной мастер сессией и сервисами EDSG.

Параметры схемы аутентификации, схемы аккаунтинга, Radius server group необходимо указывать в настройках домена, остальные параметры специфичные для абонентской сессии могут быть переданы с RADIUS сервера в стандартных или проприетарных RADIUS атрибутах.

**Значения атрибутов переданных с RADIUS всегда приоритетнее таких же настроенных локально в домене.**

### BAS интерфейс

Интерфейс маршрутизатора на котором работают BRAS/BNG функции. Обычные трафиковые политики не будут работать, только те которые на основе user-group и service-group

### Типы BAS интерфейсов и абонентских сессий

1. layer2-subscriber - VLAN, QnQ терминируется на BNG. Инициатором сессии как правило выступает DHCP Discover/ND сообщение, однако возможны варианты старта сессии по IP/ARP пакетам. BNG может выступать в качестве DHCP relay, proxy, server
1. layer3-subscriber - VLAN, QnQ терминируются на сети доступа, агрегации и на BNG приходит смаршрутизированный трафик. BNG может выступать в качестве DHCP server. Инициация сессии по IP пакету, Source IP при этом может являться частью username.
1. layer2-leased-line - VLAN, QnQ терминируется на BNG. Инициатором сессии как правило выступает DHCP Discover/ND сообщение, однако ограничение скорости и аккаунтинг трафика делается для всего интерфейса. Логин/пароль настраиваются локально на интерфейсе.
1. layer3-leased-line - VLAN, QnQ терминируются на сети доступа, агрегации и на BNG приходит смаршрутизированный трафик. BNG может выступать в качестве DHCP server, relay. Аутентификация сессии происходит по логину/паролю настроенному на сабинтерфейсе во время его перехода в состояние Operational UP. Может оказываться как сервис Интернет так и сервис доступ в MPLS L3VPN.
1. l2vpn-leased-line - VLAN, QnQ терминируется на BNG. Инициатором сессии выступает переход интерфейса в состоянии Operational UP, а аутентификация происходит по логину/паролю настроенному на сабинтерфейсе. При успешной аутентификации интерфейс включается L2VPN (VLL/VPLS) абонента.
1. Static User - особый тип абонентов, при котором аутентификация с RADIUS не нужна и абонент не обязан генерировать какой либо трафик чтобы сессия поднялась. Может быть использован для банкоматов или других специфичных кейсов.

### Методы аутентификации

1. bind - IPoE, без участия пользователя. username генерируется по информации в пакете/dhcp discover.
1. dot1x - 802.1X EAP. RFC 2284
1. ppp - PPPoE, PPPoA, пользователь явно указывает login/pass на CPE. RFC 1661.
1. web - IPoE/PPPoE + дополнительная аутентификация через Web-портал, на интерфейсах такого типа могут сосуществовать IPoE/PPPoE абоненты.
1. fast - Комбинация bind + WEB, но пользователь не вводит login/pass на Web-портале.
1. L2TP

### UNR маршруты

Особый тип маршрутов (как Static, OSPF, BGP, IS-IS) для BAS IP pool и Framed-IP адресов абонентов.

### Возможные топологии

1. Терминирование VLL/VPLS (EoMPLS, xconnect) на логическом интерфейсе BNG – Virtual-Ethernet с типом интерфейса l2-terminate и терминирование BNG абонентов на другом Virtual-Ethernet с типом интерфейса l3-access, на нем настраивается параметры BAS интерфейса. Оба интерфейса объединяются в одну VE группу, чтобы осуществить переход от L2VPN в L3VPN логически внутри оборудования. Возможен маппинг N к 1, таким образом N l2-terminate могут быть стерминированы на одном l3-terminate BAS интерфейсе. BNG выступает в качестве PE как для сети MAN, так и для IPBB сети оператора.
2. Терминирование абонентов на интерфейсе LAG участники которого могут быть распределены между несколькими линейными платами. Абоненты логически терминируется на сабинтерфейсах данной LAG. VLL используется для передачи трафика от удаленного OLT к PE, к которому непосредственно подключен BNG. Различные VLAN id определяют тот или иной VLL приходящий на PE роутер перед BNG. BNG выступает в роли CE для MAN сети и в роли PE для IPBB сети.
3. Терминирование абонентов на интерфейсе LAG участники которого распределены между несколькими линейными платами. Абоненты логически терминируется на сабинтерфейсах данной LAG. У BNG нет связности по IP/MPLS с сетью IPBB, он фактически располагается между двух IP/MPLS сетей, выступая в роли CE как для MAN сети, так и для IPBB. Сабинтерфейсы интерфейса LAG с различными VLAN служат как для терминирования абонентов, так и для связи с IPBB сетью и сетью Интернет соответственно.

### Использование ресурсов BNG

Стоит отметить что использование LAG для терминации абонентов не потребляет лишних ресурсов только если интерфейсы LAG находятся на одной LPU и обрабатываются одним и тем же NP/TM чипом, в противном случае будет тратится в X раз больше ресурсов, где X - количество NP/TM чипов задействованных в LAG.

Кроме этого абонентские лицензии на количество subscriber так же используются в Z раз больше по числу LPU в LAG.

При использовании PWHT тратятся ресурсы того слота и сабслота, который выбран на Virtual-Ethernet интерфейсе, например 1/0/Y говорит о том что используются ресурсы Slot 1, FPIC (sub-slot) 0.

### Примерная конфигурация layer2-subscriber и BIND на Eth-trunk интерфейсе(2)

#### Создание user-group и service-group локально

``` huawei_cli
#
 user-group ug-cgn
 user-group ug-access-reject
#

#
 service-group sg-world
 service-group sg-open-garden
 service-group sg-peering
 service-group sg-l4-redirect
#
```

#### Создание time-range (в данном примере не используются, но могут быть использование в UCL)

``` huawei_cli
time-range tr_night 01:0 to 8:00 daily
```

#### Создание Radius групп для аутентификации абонентов и сервисов

Можно использовать как одну и ту же так и две разных RADIUS группы для аутентификации абонентов и сервисов

``` huawei_cli
#
 radius-server source interface LoopBack0
#
radius-server group coa-source
 radius-server source interface LoopBack0
#
radius-server group users
 radius-server shared-key-cipher  authentication X.X.X.X 1812 weight 0
 radius-server shared-key-cipher  accounting X.X.X.X 1813 weight 0
#
radius-server group services
 radius-server shared-key-cipher  authentication X.X.X.X 1812 weight 0
 radius-server shared-key-cipher  accounting X.X.X.X 1813 weight 0
#
radius-server authorization X.X.X.X shared-key-cipher server-group coa-source
#
 radius-attribute hw-policy-name support-type edsg
 ```

#### Создание UCL для user-group и service-group

##### UCL для user-group и EDSG

``` huawei_cli
acl name acl-cgn-in number XXXX match-order auto
 rule 5 permit ip source user-group ug-cgn

acl name acl-open-garden number XXXX match-order auto
 rule 5 permit udp source user-group ug-access-reject destination ip-address X.X.X.X 0 destination-port eq dns
rule 10 permit udp source user-group ug-access-reject destination ip-address X.X.X.X 0 destination-port eq dns
rule 15 permit tcp source user-group ug-access-reject destination ip-address X.X.X.X 0 destination-port eq www
rule 20 permit tcp source user-group ug-access-reject destination ip-address X.X.X.X 0 destination-port eq www
rule 25 permit tcp source user-group ug-access-reject destination ip-address X.X.X.X 0 destination-port eq 443
rule 30 permit udp source service-group any destination ip-address X.X.X.X 0 destination-port eq dns
rule 35 permit udp source service-group any destination ip-address X.X.X.X 0 destination-port eq dns
rule 40 permit tcp source service-group any destination ip-address X.X.X.X 0 destination-port eq www
rule 45 permit tcp source service-group any destination ip-address X.X.X.X 0 destination-port eq www
rule 50 permit tcp source service-group any destination ip-address X.X.X.X 0 destination-port eq 443

acl name acl-opengarden-out number XXX1 match-order auto
 rule 5 permit ip source ip-address X.X.X.X X.X.X.X destination service-group sg-open-garden

#
acl name acl-peering-in number XXX3 match-order auto
 rule 5 permit ip source service-group sg-peering destination ip-address X.X.X.X X.X.X.X
rule 10 permit ip source service-group sg-peering destination ip-address X.X.X.X X.X.X.X

acl name acl-world-out number XXX4 match-order auto
 rule 5 permit ip source ip-address any destination service-group sg-world
#
acl name acl-world-in number XXX5 match-order auto
 rule 5 permit ip source service-group sg-world destination ip-address any
#
acl name acl-l4-redirect number XXX6 match-order auto
 rule 5 permit tcp source service-group sg-l4-redirect destination ip-address any destination-port eq www
rule 10 permit tcp source service-group sg-l4-redirect destination ip-address any destination-port eq 8080
rule 15 permit tcp source user-group ug-access-reject destination ip-address any destination-port eq www
rule 20 permit tcp source user-group ug-access-reject destination ip-address any destination-port eq 8080
rule 25 deny ip source service-group sg-l4-redirect
rule 30 deny ip source user-group ug-access-reject
````

#### Traffic classifier

``` huawei_cli
traffic classifier tc-cgn-in operator or
 if-match acl name acl-cgn-in
traffic classifier tc-peering-in operator or
 if-match acl name acl-peering-in
traffic classifier tc-peering-out operator or
 if-match acl name acl-peering-out
traffic classifier tc-l4-redirect operator or
 if-match acl name acl-l4-redirect
traffic classifier tc-open-garden operator or
 if-match acl name acl-open-garden
traffic classifier tc-world-in operator or
 if-match acl name acl-world-in
traffic classifier tc-world-out operator or
 if-match acl name acl-world-out
#
```

#### Traffic behaviour

``` huawei_cli
traffic behavior tb-cgn-translate
 nat bind instance nat-inst-1
traffic behavior tb-collect-stats \\поведение по умолчанию - permit
 traffic-statistic
traffic behavior tb-l4-redirect
 http-redirect
#
```

#### Traffic policy (состыковываем tc и tb)

``` huawei_cli
traffic policy tp-main-in
 share-mode
 statistics enable
 classifier tc-cgn-in behavior tb-cgn-translate precedence 2
 classifier tc-l4-redirect behavior tb-l4-redirect precedence 5
 classifier tc-open-garden-in behavior tb-collect-stats
 classifier tc-peering-in behavior tb-collect-stats
 classifier tc-world-in behavior tb-collect-stats
traffic policy tp-main-out
 share-mode
 statistics enable
 classifier tc-open-garden-out behavior tb-collect-stats
 classifier tc-peering-out behavior tb-collect-stats
 classifier tc-world-out behavior tb-collect-stats
```

#### Конфигурация traffic policy глобально к BNG, иначе функционал BNG работать не будет

``` huawei_cli
 traffic-policy tp-main-in inbound
 traffic-policy tp-main-out outbound
```

#### IP Pools (можно использовать DHCP relay/proxy тогда конфигурация немного меняется)

``` huawei_cli
ip pool public-01-pool bas local
 vpn-instance public
 gateway X.X.X.X X.X.X.X
section 0 X.X.X.X X.X.X.X
dns-server X.X.X.X X.X.X.X
lease 0 0 15
constant-index X

 ip pool-group public-01-group bas
 vpn-instance public
 ip-pool public-01-pool

```

#### Секция AAA

##### Username template и password

``` huawei_cli
aaa
default-password template default-ipoe-password cipher derparol 
default-user-name template ipoe-username-template include pevlan :  cevlan \\Определяем формат username и поля которые будут использованы для формирования

 authentication-scheme as-dynamic-customers
 authening authen-fail online authen-domain access-reject \\При отказе RADIUS абонент будет перенаправлен в соответствующий домен
  authening radius-no-response online authen-domain access-reject \\При недоступности RADIUS сервера абонент будет перенаправлен в соответствующий домен

 accounting-scheme acct
accounting interim interval 5

```

##### Конфигурация доменов (Если какие то из необходимых объектов не указаны в домене, то они должны быть переданы при аутентификации сессии)

``` huawei_cli
domain main
authentication-scheme as-dynamic-customers
accounting-scheme acct
ip-pool-group public-01-group
  user-max-session 1
  radius-server group users
 web-server url http://redirect.page.local
 web-server identical-url
  traffic match user-group inbound \\включение обработки по user-group после попадания в service-group
  ip-pool mode priority local
  undo public-address nat enable \\Не делать NAT для публичных адресов
  edsg traffic-mode rate separate statistic together

 domain access-reject
authentication-scheme none
accounting-scheme none
ip-pool-group public-01-group
  vpn-instance public
  radius-server group users
  qos-profile qp-default-shaper inbound
  qos-profile qp-default-shaper outbound
user-group ug-access-reject
web-server url http://redirect.page.local
  web-server identical-url
  edsg traffic-mode rate separate statistic together
```

Конец AAA секции

#### Указание откуда и с каким паролем аутентифицировать EDSG сервисы

``` huawei_cli
service-policy download local radius services password cipher 123456
```

#### Конфигурация BAS интерфейса

Конфигурация BAS интерфейса any-other

``` huawei_cli
interface Eth-Trunk0.1
 description B2B
 user-vlan any-other \\ Любой другой VLAN или QnQ VLANs явно не указанные на других интерфейсах. Аналог dynamic clips на E///
  bas
#
  access-type layer2-subscriber default-domain authentication main \\ указание типа интерфейса и домена по умолчанию в котором будет произведена аутентификация
  authentication-method bind \\Тип аутентификации, для IPoE - bind
  ip-trigger \\Инициация сессии по IP пакету от уже ранее аутентифицированного абонента
  arp-trigger \\Инициация сессии по ARP пакету от уже ранее аутентифицированного абонента
  default-user-name-template ipoe-username-template \\Использование username шаблона
  default-password-template default-ipoe-password \\Использование пароля для IPoE пользователей
 #
 ```

Конфигурация BAS интерфейса с явным указанием диапазона QnQ VLAN

``` huawei_cli
 interface Eth-Trunk0.2
 description blabla
 user-vlan 111 113 qinq 222 290
  bas
 #
  access-type layer2-subscriber default-domain authentication other
  authentication-method bind
  ip-trigger
  arp-trigger
  default-user-name-template ipoe-username-template
 default-password-template default-ipoe-password
#

#
```

### Анонс UNR маршрутов

``` huawei_cli
bgp 65001
 router-id X.X.X.X
 undo default ipv4-unicast
 peer X.X.X.X as-number 65001
 peer X.X.X.X connect-interface LoopBack0
 peer X.X.X.X as-number 65001
 peer X.X.X.X connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  undo peer X.X.X.X enable
  undo peer X.X.X.X enable
 #
 ipv4-family vpnv4
  policy vpn-target
  peer X.X.X.X enable
  peer X.X.X.X
 peer X.X.X.X enable
  peer X.X.X.X
#
 ipv4-family vpn-instance public
  import-route unr
 #
 ```

### Пример конфигурации сервисов в FreeRadius

С точки зрения конфигурации FreeRadius и процесса аутентификации аутентификация сервиса ничем не отличается от аутентификации абонента

``` radius_config
############################################################
# 1. Назначение двух сервисов                              #
############################################################
"s_open"  Cleartext-Password:="123456"
        HW-AVpair = "service:service-group=sg-open-garden priority 100",
        HW-AVpair += "service:authentication-scheme=none",
        HW-AVpair += "service:accounting-scheme=acct",
        HW-AVpair += "service:radius-server-group=users",
        HW-Input-Committed-Information-Rate = "2000000",
        HW-Output-Committed-Information-Rate += "2000000"
        HW-Priority = "60"

"s_inet" Cleartext-Password := "123456"
        HW-AVpair = "service:service-group=sg-world priority 200",
        HW-AVpair += "service:authentication-scheme=none",
        HW-AVpair += "service:accounting-scheme=acct",
        HW-AVpair += "service:radius-server-group=users",
        HW-Input-Committed-Information-Rate = "3000000",
        HW-Output-Committed-Information-Rate += "3000000"

"s_2inet" Cleartext-Password := "123456"
        HW-AVpair = "service:service-group=sg-world priority 200",
        HW-AVpair += "service:authentication-scheme=none",
        HW-AVpair += "service:accounting-scheme=acct",
        HW-AVpair += "service:radius-server-group=users",
        HW-Input-Committed-Information-Rate = "8000000",
        HW-Output-Committed-Information-Rate += "8000000"

"s_multi" Cleartext-Password := "123456"
        HW-AVpair = "service:service-group=sg-world priority 200",
        HW-AVpair += "service:authentication-scheme=none",
        HW-AVpair += "service:accounting-scheme=acct",
        HW-AVpair += "service:radius-server-group=users",
        HW-Input-Committed-Information-Rate = "5000000",
        HW-Output-Committed-Information-Rate += "5000000"
```

Абонент с QnQ VLAN s-vlan 1001 c-vlan 2001 и двумя EDSG сервисами и HW IP Pool группой = public-01-group

``` raduis_config
1001:2001@main  Cleartext-Password := "derparol"
        Service-Type = Login-User
        HW-Account-Info = "As_open",
        HW-Account-Info += "As_inet"
        HW-Framed-Pool-Group = "public-01-group",
```

### Huawei AV-Pair для сервисов и не только

Больше информации про HW-AVpair (188) можно найти в [документации](https://support.huawei.com/hedex/hdx.do?lib=EDOC1000144075AEI0605X&docid=EDOC1000144075&lang=en&v=05&tocLib=EDOC1000144075AEI0605X&tocV=05&id=AEI0605X_05_19676&tocURL=resources%252fme60%255fpublic%255fen%252fne%252fdc%255fne%255fcfg%255f013537a%252ehtml&p=t&fe=1&ui=3&keyword=method&keyword=authent&text=Configuring%25252BWeb%25252B%2525253Cb%2525253EAuthentication%2525253C%2525252Fb%2525253E%25252Bor%25252BFast%25252B%2525253Cb%2525253EAuthentication%2525253C%2525252Fb%2525253E)

Huawei RADIUS атрибуты которые могут понадобиться

* HW-Input-Committed-Information-Rate
* HW-Output-Committed-Information-Rate
* HW-Input-Committed-Burst-Size
* HW-Output-Committed-Burst-Size
* HW-Subscriber-QoS-Profile (для локальных QoS profile)
* HW-Data-Filter
* HW-Command-Mode
* HW-VPN-Instance

AVPair относящиеся к EDSG на которые стоит обратить внимание.

* service:service-group
* service:authentication-scheme
* service:accounting-scheme
* service:radius-server-group
* service:time-range
* subscriber:traffic-policy-in
* subscriber:traffic-policy-out

Для особо отважных рекомендую разобраться в [More Information About HW-Data-Filter (82)](https://support.huawei.com/hedex/hdx.do?lib=EDOC1000144075AEI0605X&docid=EDOC1000144075&lang=en&v=05&tocLib=EDOC1000144075AEI0605X&tocV=05&id=AEI0605X_05_19676&tocURL=resources%252fme60%255fpublic%255fen%252fne%252fdc%255fne%255fcfg%255f013537a%252ehtml&p=t&fe=1&ui=3&keyword=method&keyword=authent&text=Configuring%25252BWeb%25252B%2525253Cb%2525253EAuthentication%2525253C%2525252Fb%2525253E%25252Bor%25252BFast%25252B%2525253Cb%2525253EAuthentication%2525253C%2525252Fb%2525253E) который позволяет передавать динамически UCL на основе service-group, а так же создавать динамически traffic classifier и traffic behaviour.

Для примера следующее содержание  HW-Data-Filter `
RC=class1;SS-GROUP=google;DIP=1;1.1.0.0/16;DIP=2.2.0.0/16;DIP=3.3.0.0/16;BI-DIR; RB=behavior1@RB=behavior1;permit;` эквиваленто созданию TC = class1, Behavior = behavior1 и UCL c использованием service-group = google приведенного ниже локально на BNG.

``` huawei_cli
rule permit ip source ip-address 1.1.0.0 0.0.255.255 destination service-group google
rule permit ip source service-group google destination ip-address 1.1.0.0 0.0.255.255
rule permit ip source ip-address 2.2.0.0 0.0.255.255 destination service-group google
rule permit ip source service-group google destination ip-address 2.2.0.0 0.0.255.255
rule permit ip source ip-address 3.3.0.0 0.0.255.255 destination service-group google
rule permit ip source service-group google destination ip-address 3.3.0.0 0.0.255.25
```

**Однако данный метод имеет серьезный недостаток из-за максимального размера сообщения в RADIUS так как RADIUS работает поверх UDP, а имплементировать [RADIUS over TCP](https://tools.ietf.org/html/rfc6613) никто из вендоров BNG особо не желает.**

### Бонус

1. Для траблшутинга юзеров есть команда `trace access-user`, выводит более чем полную информацию о процессах аутентификации, получения IP и прочего.
1. Кэш сервисов может быть обновлен вручную или по таймеру. Настраивается командой `service-policy cache update interval interval-value`
1. Возможна настройка Hot-backup абонентских сессий при наличии двух BNG. Абонентские сессии синхронизируются между двумя BNG. Балансировка возможно по MAC адресам. Разобраться можно самому, но тяжело. Hot-backup + EDSG работает в V6R9C20.
1. BNG умеет определять доступность терминала через arp-ping и определять "живость" сессии по количеству трафика в промежуток времени `user detect retransmit num interval time [ no-datacheck ]`

``` huawei_cli
trace enable
trace access-user object object-id { access-mode { pppoe | pppoa | pppoeoa | ipoeoa | ipoe } | user-name username | tunnel-id tunnel-id | interface interface-type interface-number [ pvc vpi/vci ] | ip-address ip-address | mac-address mac-address | ce-vlan ce-vlan-id | pe-vlan pe-vlan-id | ipv6-address ipv6-address/prefixlength } * [ output { file file-name | syslog-server ip-address | vty } | [ -t time ] | [ mode packet ] ] *
```
