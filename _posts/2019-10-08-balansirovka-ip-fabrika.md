---
layout: post
title: "Особенности балансировки в EVPN/VXLAN фабрике"
tags: arista vxlan bgp evpn clos lacp
author: "ipotech"
---


> **Disclaimer**:
Тут не описаны все особенности, это не претендует на роль серебряной пули, тут только то, с чем столкнулся автор и какие решения применил


## Причина

В [статье](/_posts/2019-06-25-servisnaja-arhitektura-isp.md) я привел схему того, как можно объединить площадки одного провайдера в городе, за недорого, с использованием ДЦ телекоммуникационного оборудования. Так же, там я упомянул про возможные подводные камни по балансировке трафика (а как мы помним, фабрика передает MPLS трафик, да еще и для [B2B](/_posts/2019-06-24-juniper-mx-mpls-examples.md)) включенных маршрутизаторов.


## Исходные данные

Arista 7280SR для bridged MPLS имеет в балансировке два сценария:
- local bridged: тогда Outer Eth + MPLS;
- VXLAN decap: тогда Outer IP/UDP + Inner Eth.



Так же вывод с самой коробки:

```
#show load-balance profile
default (global):
Lag Hashing on IP-TCP-UDP headers for IP packets is ON
Lag Hashing on MAC header for IP packets is ON
Full symmetric hashing is OFF
Lag Hashing mode is flow-based
Lag Hash seed is 0 (automatically selected)*
Lag Hash polynomial is 3 (automatically selected)*
Lag Hash key shift is 16 (automatically selected)*
Lag Hashing on ingress interface is ON
Do not adjust the starting header on switchport is OFF
Allowed member selection methods: Multiplication SmoothDistribution
Port-channel load-balancing in egress replication is OFF
MAC hash fields:
   Source MAC Address is ON
   Destination MAC Address is ON
   EtherType is ON
   VLAN is ON
MPLS hash fields:
   Label is ON
   Entropy label is OFF
IPv4 hash fields:
   Source IPv4 Address is ON
   Destination IPv4 Address is ON
   Protocol is ON
   Time-To-Live is OFF
IPv6 hash fields:
   Source IPv6 Address is ON
   Destination IPv6 Address is ON
   Flow Label is ON
   Hop Limit is OFF
   Next Header is ON
L4 hash fields:
   Source Port is ON
   Destination Port is ON
Packet type MPLS over GRE:
   Hashing mode is inner-ip

```


### Первый этап
Вооружившись этими знаниями мы решили стартовать переключение устройств со 100G портами PE-LEAF, а именно их было по 2 в каждый LEAF. Так же, мы настроили между этими PE и до всех PE в старой сети по 4 UHP LSP (всего порядка 40 LSP или 40 меток).

В итоге мы получили вот такую картинку:

![stage_1](/images/arista_stage_1.png):

Паттерн трафика был от A0B в A1q к старой сети, трафик в обратном направлении был небольшим.


**Точки балансировки:**
- Juniper A0B без проблем балансирует по 4 100G интерфейсам;
- Arista смотрит на MPLS заголовок и отправляет трафик:
  - или локально (в один порт 100G сторону A1Q) - ожидаемо без проблем;
  - или в сторону удаленного LEAF. И так как доля трафика между самими A0B не велика, а старая сеть - это много маршрутизаторов, то количество MPLS меток было достаточным для красивой (не совсем идеальной, но очень приемлемой) балансировки на участке LEAF-SPINE-LEAF.

![balance](/images/arista_stage_1_1.png)

Все выглядело хорошо и мы переключили все "одноногие" (по одному порту в leaf) PE


### Второй этап
Одним из преимуществ схемы считалось то, что мы можем подключать PE к LEAF LAG из 10G портов, которые, как известно, стоят очень и очень не дорого (да речь про MPC-3D-16XGE-SFPP)

Так мы и поступили, на этих же площадках как раз было по такому устройству. В итоге схема уже имела вот такой вид:

![stage_2](/images/arista_stage_2.png)

Но сфокусируем внимание на подключении А2-17, А2-18 и шаблон их трафика:

![stage_2_1](/images/arista_stage_2_1.png)

Итак:
- A2-17 и A2-18 включены 16*10g портами в LEAF;
- Между A0B и A2* прописано по 4 LSP;
- 90% трафик это один MPLS L3VPN, то есть одна сервисная метка;
- Трафик идет от A0B к A2, при этом внутри площадки остается до 90% трафика. В обратном направлении трафика мало - в расчет не берем;
- Трафик между А2 есть (как и 4 LSP), но его очень мало, в расчет не берем;
- L3 связь между PE одна, то есть одна пара мак и IP адресов.

**Точки балансировки:**
- Juniper A0B без проблем балансирует по 4 100G интерфейсам;
- Arista смотрит на MPLS заголовок и отправляет трафик:
  - в сторону удаленного VTEP - тут по прежнему все хорошо;
  - локально на 8 портов в LACP.

И вот что мы получили на egress линка LEAF-A2:

![stage_2_1](/images/arista_stage_2_2.png)

И это еще сохранился удачный скриншот, на некоторых LEAF все было сильно хуже. Одна десятка в полке, а две с 700Mb/s

Балансировка Juniper по 8 LSP:

![stage_2_3](/images/arista_stage_2_3.png)

Нужно было что-то быстро менять. Первое что мы сделали это увеличили количество LSP до 8, затем до 16. Становилось лучше, но все равно был перекос. Один линк при этом так и остался в полке на ЧНН, но тюнинг большого буфера помог пережить ночь.

Так же есть факт что Juniper выделяет метки по порядку. Но случайный клиринг LSP в качестве попытки улучшить результат hash ни к чему хорошему не приводили.

Опираясь на опыт первого этапа была идея решить проблему путем уменьшения количество портов в LAG группе. Сделать это можно было так:
- между двумя PE поднималась еще одна связь в отдельном VLAN;
- на фабрике разные VLAN спускались в разные LAG группы.

В этой схеме juniper балансировал по двум ECMP IGP маршрутам 16 LSP. Сделав это мы получили улучшение:

![stage_2_4](/images/arista_stage_2_4.png)

Но  результат был все равно не такой, как нам нужно. Как вариант, можно было довести количество таких интерфейсов до количества портов, то есть до 8, но сами понимаете какие тут возникают проблемы в эксплуатации.

![shtosh](/images/shtosh.jpg)

Казалось что схема не взлетит.


**НО!**
Благодаря активной помощи со стороны инженеров Arista и их общению с разработчиками удалось найти решение. И про него даже оказалась статья в базе [знаний](https://eos.arista.com/eos-4-20-5f/vxlan-o-mpls-lag-hashing-optimizations-for-l2-ports/):

![stage_2_5](/images/arista_stage_2_5.png)

Эта "кнопка" сделала сдвиг на один заголовок, а именно на первый ETH, это позволило заглянуть в IP заголовок за MPLS заголовком, а так как там многообразие интернета, то балансировка тут же стала очень ровной.


**HAPPY END**
