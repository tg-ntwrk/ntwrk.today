---
layout: post
title:  "Интернет в VRF vs Интернет в GRT"
tags: internet vrf grt holywar
author: "riddler63"
---

Интернет в VRF vs Интернет в GRT

Существует давний холивар по поводу того где хранить/распространять Internet FullView/FullFeed, в VRF или в GRT. В данной статье я попробовал собрать все за и против Интернета в VRF не особо углубляясь в детали.

## Pros

1. BGP free core из коробки. Возможно и при Интернет в GRT если аллоцировать метки и для /30, /31,  /28 p2p сетей или при использовании iBGP NHS.
2. Оптимальное использование RR при уникальных RD, как следствие prefix visibility и оптимальные пути не зависящие от иерархии RR.
   1. Похожий функционал при Internet в GRT можно получить используя BGP add-path или RR Optimal Route Reflection
3. [Route oscillation](https://tools.ietf.org/html/rfc3345) так же решается разными RD, в то время как в GRT необходимо использовать [ADD-PATH](https://tools.ietf.org/html/rfc7911) c [Advertise the Group Best Paths]( (https://tools.ietf.org/html/rfc7964#section-4))
4. Легкое и гибкое разделение клиентов на:
    1. Конечных потребителей
    2. Транзит
    3. IXP
    4. Другие
5. Легкое и гибкое распространение всех или части маршрутов на основе политик и RT
6. Защита от DDoS (легко завернуть трафик) импортировав специфик где надо по RT.
7. Простая изоляция кастомеров
8. RTC/RTF - фильтрация ненужных префиксов еще на RR/Peer без приёма оных в BGP Adj-RIB-in.
   1. Возможность давать кастомерам интернет даже на слабых железках, default + partial full feed при этом сохраняя оптимальность прохождения трафика. Согласно докладу на NANOG76 95% трафика генерируют < 500 префиксов. [Internet Traffic 2009 2019](https://www.youtube.com/watch?v=jGnVcCQUCdk&feature=youtu.be)
9. Возможность отдавать отдельно страну, мир или их микс (специфично для Украины и других стран)
10. Чёткое разделение между внутренней сетью ISP и публичным Internet
11. Наличие 6VPE. В Internet in VRF надо настраивать 6PE (BGP-LU) чтобы прогнать IPv6 трафик по MPLS BGP free core.
12. Возможность использовать BGP PIC Edge  

## Cons

1. бОльшее потребление памяти в RIB (+8%) + [~80% памяти за каждый VRF с FV](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)
2. бОльшее потребление памяти в FIB на X%. Однако на некотором оборудовании FIB структуры для GRT такие же как и для VRF.
3. Возможная недоступность технологий безопасности BGP (e.g RKPI, BGP FlowSpec + other BGP security features) в VRF у некоторых вендоров.
4. Использование RTC/RTF ухудшает сходимость
5. При падении p2p линка c eBGP peer на PE никаким другим образом, кроме как сигнализацией через BGP, нельзя вызвать пересчёт таблиц маршрутизации на удалённых PE. Для сравнения при наличии Интернета в GRT можно включить p2p линки в IGP, IGP сходится, как правило, гораздо быстрее, но сильно зависит от домена IGP.
6. Всевозможные грабли, из-за того что вендоры (привет J,H,N,C) не реализовали фичу X в VRF.
7. При mpls label per route метки кончаются сразу
8. При mpls label ver VRF роутер Egress PE дополнительно делает IP lookup

Ссылки:

1. [Peering Fabric Design -> Peering and Internet in a VRF](https://xrdocs.io/design/blogs/latest-peering-fabric-hld#security)
2. [Is it safe to run Internet in a VRF?](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)
3. [internet routing table in a vrf](https://seclists.org/nanog/2013/Mar/154)
4. [Service Provider Network Architecture - Internet in a VRF](https://www.reddit.com/r/networking/comments/5j8vg8/service_provider_network_architecture_internet_in/)
5. [Internet Traffic 2009 2019](https://www.youtube.com/watch?v=jGnVcCQUCdk&feature=youtu.be)
6. [RFC ADD-PATH](https://tools.ietf.org/html/rfc7911)
7. [Advertise the Group Best Paths]( (https://tools.ietf.org/html/rfc7964#section-4))