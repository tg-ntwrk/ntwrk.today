---
layout: post
title: "Особенности работы ARP протокола в Junos Network Operating System"
tags: juniper arp policer ddos
author: "orlik"
---

Особенности обработки ARP запросов в Junos OS, а также защита от атак с использованием протокола ARP.

## Обработка исходящих ARP запросов 

Когда на маршрутизатор приходит пакет, у которого destination адрес принадлежит direct сети на ethernet интерфейсе, Juniper PFE выполняет поиск в таблице маршрутизации для адреса назначения, результатом которого возможны три варианта:

### NH Type "Unicast"

Будет найдена /32 запись с NH Type "Unicast", это означает, что для данного маршрута уже есть ARP запись и пакет можно отправить далее  

```
> request pfe execute command "sh route ip prefix 10.0.150.205 detail" target afeb0
SENT: Ukern command: sh route ip prefix 10.0.150.205 detail

IPv4 Route Table 0, default.0, 0x80000:
Destination        NH IP Addr      Type     NH ID Interface
------------------ --------------- -------- ----- ---------
10.0.150.205       10.0.150.205    Unicast   986  RT-ifl 444 ge-0/0/1.172 ifl 444
```
### NH Type "Resolve"

Будет найдена запись с NH Type "Resolve". Подобная запись является конечной в поиске только при отсутствии /32 записей для данного адреса. Когда трафик попадает под Resolve маршрут, [Lookup chip](https://null.53bits.co.uk/index.php?page=mx-trio-pfe-lu-deep-dive) <sup id="a1">[1](#f1)</sup> отправляет оповещение на PFE CPU о необходимости выполнить резолв для данного адреса. Создается локальная запись о том, что FPC пытается резолвить данный маршрут, затем отправляется запрос на RE о необходимости выполнения резолва. 
```
> request pfe execute command "sh route ip prefix 10.0.150.200/29" target afeb0
SENT: Ukern command: sh route ip prefix 10.0.150.200/29


IPv4 Route Table 0, default.0, 0x80000:
Destination        NH IP Addr      Type     NH ID Interface
------------------ --------------- -------- ----- ---------
10.0.150.200/29                    Resolve   968  RT-ifl 444 ge-0/0/1.172 ifl 444
```
Если по истечении этого таймера, от RE не будет получено ответа (например, хост недоступен и не может отрезолвлен), то интерфейс, с которого пришел запрос на резолв, на 10 секунд помечается как throttled и все дельнейшие запросы о необходимости резолва от данного интерфейса и приходящие на FPC CPU игнорируются. При этом можно видеть в syslog FPC сообщения вида
```
[Jan  1 01:01:01.001 LOG: Info] NH_RESOLUTION_REQ_THROTTLED: Next-hop resolution requests from interface 350 throttled
```

### NH Type "Hold"

Третий вариант записи в таблице имеет NH Type "Hold". Данный тип записи создает RE после того, как получит запрос на резолв от FPC CPU, после чего RE отправляет ARP who-has запрос с интерфейса (в примере это ge-0/0/1.172). Данный тип записи необходим, чтобы избежать ситуации когда RE одновременно отправит несколько запросов на резолв для одного и того же адреса (например, пришедшие с разных FPC).

```
> request pfe execute command "sh route ip prefix 10.0.150.204" target afeb0
SENT: Ukern command: sh route ip prefix 10.0.150.204


IPv4 Route Table 0, default.0, 0x80000:
Destination        NH IP Addr      Type     NH ID Interface
------------------ --------------- -------- ----- ---------
10.0.150.204                        Hold    9705  RT-ifl 444 ge-0/0/1.172 ifl 444
```

Если RE получит ответ на свой ARP запрос, то запись будет сконвертирована из Hold в Unicast, в ARP таблице появится соответствующая запись.

```
> show arp no-resolve interface ge-0/0/1.172 expiration-time
MAC Address       Address         Interface                TTE    Flags
00:0c:29:0a:37:7f 10.0.150.205   ge-0/0/1.172             1441    none
Total entries: 1
```

## Обработка входящих ARP запросов

Все входящие ARP пакеты попадают под действие [`__default_arp_policer__`](https://www.juniper.net/documentation/en_US/junos/topics/concept/security-arp-policer-overview.html)  <sup id="a2">[2](#f2)</sup>. Данный полисер применен глобально на каждом PFE и обрабатывает arp запросы для всех интерфейсов одного PFE.
> **Примечание**: Учитываются не только запросы с broadcast адресом назначения, но и все пакеты, у которых EtherType = 0x0806, в том числе ответы на запросы, отправленные маршрутизатором. И даже пакеты с Dst MAC других клиентов, которые попали на маршрутизатор ошибочно, например из-за петли или hash collision на L2 оборудовании, подключенном на данном интерфейсе.

Пропускная способность `__default_arp_policer__` - 150000 Bps, burst - 15000 bytes. При размере ARP запроса в 42 байта данный полисер может пропустить ~446 pps. Если на один PFE поступает больше 446 пакетов в секунду, все лишние запросы будут отброшены полисером. Cуществует вероятность, что часть пакетов не дойдет до RE, как следствие, ARP записи на интерфейсах этого PFE не обновятся и будут удалены. Это черевато кратковременными пропаданиями связи у клиентов, и, как вариант, падением некоторых протоколов маршрутизации (BGP/LDP/RSVP).

Все пакеты далее попадают на PFE CPU, откуда пересылаются на RE. 

Между PFE и RE пакеты попадают в отдельную очередь `L2 Packets`
```
> request pfe execute command "show ttp statistics" target afeb0
...
TTP Receive Statistics:
                   Control        High      Medium         Low     Discard
                ----------  ----------  ----------  ----------  ----------
 L2 Packets              0         129          80           0           0
 L3 Packets              0         754        1375           0           0
 Drops                   0           0           0           0           0
 Queue Drops             0           0           0           0           0
 Unknown                 0           0           0           0           0
 Coalesce                0           0           0           0           0
 Coalesce Fail           0           0           0           0           0
...
```

После чего все ARP пакеты от всех PFE попадают в kernel очередь, где и обрабатываются. 

## Полисер `__default_arp_policer__`

Отслеживать `__default_arp_policer__` полисер возможно из CLI:
```
> show policer __default_arp_policer__
Policers:
Name                                                Bytes              Packets
__default_arp_policer__                       64833769988           1439659193
```

или по SNMP (лучше по pps)
```
OID: .1.3.6.1.4.1.2636.3.5.1.1.4.23.95.95.100.101.102.97.117.108.116.95.97.114.112.95.112.111.108.105.99.101.114.95.95.23.95.95.100.101.102.97.117.108.116.95.97.114.112.95.112.111.108.105.99.101.114.95.95
```
Также возможно использовать телеметрию. 

## Способы защиты от ARP flood

Полисер `__default_arp_policer__` необходим для защиты RE от флуда. Если не отслеживать количество отброшенных им пакетов, то при флуде от клиентов маршрутизатор будет функционировать стабильно, в отличии от клиентов, которые будут испытывать серьезные проблемы со связью. Существует несколько вариантов защиты от подобных ситуаций. 

### Индивидуальные ARP полисеры
На каждый логический интерфейс примененяется индивидуальный ARP полисер и все ARP пакеты на этом интерфейсе не будут попадать под действие `__default_arp_policer__`. В случае флуда на данном интерфейсе все лишние пакеты будут отброшены, не оказывая никакого влияния на другие интерфейсы.

```
> show configuration interfaces ge-0/0/1.172
family inet {
    policer {
        arp arp8k;
    }
    address 10.0.150.201/29;
}
```
### Отключение `__default_arp_policer__`
Полисер `__default_arp_policer__` отключается на каждом логическом интерфейсе физического интерфейса командой
```
set interfaces ge-0/0/1.172 family inet policer disable-arp-policer
```
> **Внимание**: В этом случае на PFE нет никакой защиты от ARP флуда и большое количество запросов может повлиять на стабильность системы!

Проверить отключение полисера возможно по отсутствию строки
```
Policer: Input: __default_arp_policer__
```
в выводе команды
```
show interface detail ge-0/0/1.172
```

### DDoS Protection
В Junos OS присутствует механизм [DDoS protection](https://www.juniper.net/documentation/en_US/junos/topics/concept/subscriber-management-ddos-protection.html) <sup id="a3">[3](#f3)</sup> для защиты RE от всевозможных атак, в том числе и от флуда ARP пакетами. Особенностью реализации данного вида защиты является очередность, с которой обрабатывается ARP трафик:

Cначала трафик попадает под действие ARP полисера (индивидуального или `__default_arp_policer__`), затем обрабатывается системой анализа DDoS protection.

Таким образом, с настройками по умолчанию (bandwidth 20000 pps) DDoS protection для протокола ARP никогда не сработает. Необходимо снижать пороговые значения для того, чтобы система детектировала атаки. 

## Комбинированный пример
Рассмотрим случай:
* Отсутствие каких-либо полисеров для ARP запросов на интерфесах;
* Настроенный DDoS protection для протокола ARP, чтобы он срабатывал раньше, чем пакеты начнут отбрасываться `__default_arp_policer__` (bandwidth 400 pps, burst 100 pps);
* Включен flow-detection, хотя данное условие не является обязательным;
* От всех клиентов поступает ~100 pps ARP пакетов;
* Клиент, от которого поступает большое количество ARP запросов: ~1100 pps.

 Все ARP запросы попадут под действие `__default_arp_policer__`, после которого пройдет только ~440 pps. Эти 440 pps вызовут срабатывание DDoS protection и включение flow-detection. Вредоносные flow будут заблокированы и не поступят на Control Plane оборудования.
 
 Но блокироваться данные вредоносные flow будут после `__default_arp_policer__`, таким образом, из 1100 pps более 600 pps будут по-прежнему отбрасываться на `__default_arp_policer__`, что будет негативно сказываться на работе других клиентов.

Поэтому при срабатывании DDoS-protection для восстановления нормальной работы клиентов, необходимо выставлять индивидуальные ARP полисеры для интерфейсов "проблемных клиентов". Это возможно автоматизировать через конфигурирование event-options, например:
```
policy arpddos {
    events ddos_scfd_flow_found;
    attributes-match {
        ddos_scfd_flow_found.protocol-name matches ARP;
    }
    then {
        change-configuration {
            commands {
                "set interfaces {$$.source-name} family inet policer arp arp8k";
            }
        }
    }
}
```

Также возможно выполнение скрипта, который будет применять ARP полисеры различной емкости в зависимости от сконфигурированной на интерфейсе маски сети.

## Ссылки
<b id="f1">1</b>. [MX Series LU-chip Overview
](https://null.53bits.co.uk/index.php?page=mx-trio-pfe-lu-deep-dive) [↩](#a1)<br/>
<b id="f2">2</b>. [ARP Policer Overview](https://www.juniper.net/documentation/en_US/junos/topics/concept/security-arp-policer-overview.html) [↩](#a2)<br/>
<b id="f3">3</b>. [Control Plane Distributed Denial-of-Service (DDoS) Protection Overview](https://www.juniper.net/documentation/en_US/junos/topics/concept/subscriber-management-ddos-protection.html) [↩](#a3)<br/>

