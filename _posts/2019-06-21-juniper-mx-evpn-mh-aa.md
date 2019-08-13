---
layout: post
title: "Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX"
tags: juniper mpls bgp evpn l3vpn
author: "somovis"
---

Самое необходимое про EVPN MH A/A + L3VPN на Juniper MX

## Коротко о EVPN

EVPN использует новое BGP NLRI, именуемое EVPN NLRI. Это NLRI передаётся в существующем AFI 25 \(L2VPN\) и новом SAFI 70 \(EVPN\).

Проблемы VPLS или что было до EVPN:

- В сценарии multihoming \(далее MH\) возможен только Active/Standby;
- Все виды VPLS не предоставляют расширенных возможностей L3, за исключением добавления BVI/IRB интерфейса в VPLS домен для Inter VLAN Routing, вследствие чего на оборудовании \(зависит от реализации\) выполняется больше операций lookup и возможен неоптимальный Packet Flow;
- MAC-адреса изучаются исключительно на уровне Data Plane, что приводит к увеличению флуда BUM трафика в сети;
- Re-learn и частичная потеря трафика во время VMotion.

Решения проблем, которые предлагает EVPN:

- В сценарии MH для каждого Ethernet Segment \(далее ES\) возможно использовать
  - Active/Standby \(A/S\);
  - Active/Active \(A/A\);
- У EVPN нет разных типов, что облегчает понимание технологии, поиск и устранение неисправностей, а также EVPN легко расширяется за счёт добавления новых [Route Types](http://bgphelp.com/2017/05/22/evpn-route-types/), но не все Route Types и возможности EVPN поддерживаются разными производителями и/или аппаратными/программными платформами;
- EVPN предоставляет расширенные возможности для Inter VLAN Routing, имеет оптимальный Packet Flow и большой набор возможностей для DCI;
- MAC-адреса изучаются на уровне Control Plane \(поведение больше похоже на L3VPN, только не по отношению к IP-адресам, а к MAC-адресам\), это позволяет получить оптимальный Packet Flow и уменьшить BUM трафик в сети;
- В Junos OS 18.4 появилась поддержка [VMTO](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-ingress-vmto.html), что позволяет оптимизировать входящий трафик на шлюз по умолчанию из WAN во время VMotion. Достигается это \(в идеальном примере\) средствами [composite next hop](https://habr.com/ru/post/324268/).

## Чем хорош EVPN MH, почему именно A/A и причём тут L3VPN

Чтобы не быть переводчиком, рекомендую прочитать [RFC](https://tools.ietf.org/html/rfc7432) и [реализацию Juniper](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html), а русскоязычным пользователям для более детального ознакомления [статью на habr](https://habr.com/ru/post/316792/).

Краткое описание:

- EVPN MH полезен тем, что можно использовать более одного PE маршрутизатора за каждый ES для обеспечения резервирования ES;
- EVPN MH A/A полезен тем, что каждый PE маршрутизатор за каждый ES способен отправлять и принимать трафик. Это позволит использовать Load Sharing по всем интерфейсам всех PE-CE. К сожалению, это приводит к усложнению поиска и устранения неисправностей;
- L3VPN необходим для обеспечения связности между L3VPN-only маршрутизаторами и маршрутизаторами с поддержкой EVPN + L3VPN.

## А теперь соберем всё вместе

До Junos OS 18.4 при использовании EVPN MH A/A + L3VPN:
- EVPN route type 2 MAC+IP анонсировались со всех PE за ES, но в L3VPN Host Prefix \(на Juniper по умолчанию при добавлении интерфейса IRB в EVPN RI и VRF RI импортируются префиксы из EVPN Route Type 2 MAC+IP в VRF как /32 префиксы\) анонсировались только с того PE маршрутизатора, на котором первым был изучен ARP \(EVPN Route Type 2 MAC+IP изучаются в напрямую подключенном ES при получении ARP\), что, в свою очередь, не позволяло сделать A/A для всего входящего L3VPN-only трафика из WAN за этот ES.

Начиная с Junos OS 18.4 при использовании EVPN MH A/A + L3VPN:
- Добавили [Multihomed Proxy MAC and IP Address Route Advertisement](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html#jd0e651), что позволило анонсировать EVPN Route Type 2 MAC+IP и L3VPN Host Prefix со всех PE маршрутизаторов той площадки, на которой были изучены EVPN Route Type 2 MAC+IP в напрямую подключенном ES. Опционально можно добавить VMTO и нехитрую политику \(политику по желанию\).

Данный сценарий был проверен с успешным результатом на оборудовании Juniper MX204 18.4R2 и на данный момент используется в производстве.

## Топология

![Topology](/images/juniper-mx-evpn-mh-aa-topology.png)

## Пример конфигурации

Конфигурация может изменятся по мере реализации новых возможностей производителем.

> **Примечание:** Часть важных опций прокомментирована.

```
version 18.4R2.7;
# Для сокращения строк текста конфигурации.
groups {
    interface-evpn-all-active {
        interfaces {
            <*> {
                unit <*> {
                    encapsulation vlan-bridge;
                    # vESI - обязательный параметр. Так же позволяет сократить потери трафика во время ручного обслуживания (unit disable).
                    esi {
                    # Автоматическое наследование ESI от локального LACP system-id.
                        auto-derive {
                            lacp;
                        }
                        all-active;
                    }
                }
            }
        }
    }
    ri-vrf {
        routing-instances {
            <*> {
                instance-type vrf;
                vrf-table-label;
                routing-options {
                    protect core;
                    auto-export;
                }
            }
        }
    }
    ri-evpn {
        routing-instances {
            <*> {
                instance-type evpn;
                protocols {
                    evpn {
                        # Используется по умолчанию. Уменьшает утилизацию LFIB/FIB, но потенциально ухудшает скорость сходимости в сравнении с per-mac.
                        label-allocation per-instance;
                        # Синхронизация шлюза в конкретном примере не нужна т.к. MAC/IP на всех PE заданы одинаковые.
                        default-gateway do-not-advertise;
                    }
                }
            }
        }
    }
}
system {
    host-name pe01;
    no-redirects;
    # Потенциально позволяет повлиять на изучение MAC+IP и как следствие хост маршрутов.
    arp {
        aging-timer 5;
        passive-learning;
        purging;
        gratuitous-arp-on-ifup;
        gratuitous-arp-delay 3;
    }
}
chassis {
    aggregated-devices {
        ethernet {
            device-count 1;
        }
    }
    # Обязательный параметр. Требуется перезагрузка FPC после изменения.
    network-services enhanced-ip;
}
interfaces {
    et-0/0/2 {
        description "link with fb";
        gigether-options {
            802.3ad ae0;
        }
    }
    et-0/0/3 {
        description "link with fa";
        gigether-options {
            802.3ad ae0;
        }
    }
    ae0 {
        flexible-vlan-tagging;
        mtu 9192;
        encapsulation flexible-ethernet-services;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                # Обязательный параметр. Должен быть задан уникальным в пределах ES на всех PE.
                system-id 0a:00:01:01:00:00;
            }
        }
        unit 3550 {
            apply-groups interface-evpn-all-active;
            vlan-id 3550;
        }
    }
    irb {
        unit 3550 {
            family inet {
                # Должен быть задан одинаковым в пределах RI на всех PE.
                address 10.50.50.254/24;
            }
            # Должен быть задан одинаковым в пределах RI на всех PE.
            mac 0a:00:00:00:35:50;
        }
    }
}
forwarding-options {
    load-balance {
        indexed-load-balance;
        per-flow {
            hash-seed;
        }
    }
    hash-key {
        family inet {
            layer-3;
            layer-4;
        }
        family mpls {
            label-1;
            label-2;
            label-3;
        }
        family multiservice {
            source-mac;
            destination-mac;
        }
    }
    enhanced-hash-key {
        family inet {
            incoming-interface-index;
            type-of-service;
        }
        family mpls {
            label-1-exp;
            incoming-interface-index;
        }
        family multiservice {
            incoming-interface-index;
        }
    }
}
policy-options {
    policy-statement Load_balance {
        term 1 {
            then {
                load-balance per-packet;
            }
        }
    }
    # Позволяет назначить приоритеты анонсов для определенных типов маршрутов.
    policy-statement evpn-rt-priority-policy {
        term 1 {
            from {
                family evpn;
                nlri-route-type 1;
            }
            then {
                bgp-output-queue-priority priority 15;
            }
        }
        term 2 {
            from {
                family evpn;
                nlri-route-type 2;
            }
            then {
                bgp-output-queue-priority priority 11;
            }
        }
        term 3 {
            from {
                family evpn;
                nlri-route-type 3;
            }
            then {
                bgp-output-queue-priority priority 12;
            }
        }
        term 4 {
            from {
                family evpn;
                nlri-route-type 4;
            }
            then {
                bgp-output-queue-priority priority 15;
            }
        }
        term 5 {
            from {
                family evpn;
                nlri-route-type 5;
            }
            then {
                bgp-output-queue-priority priority 11;
            }
        }
    }
    policy-statement vrf-x_export {
        term vip {
            from {
                # Префикс, который мы будем импортировать на L3VPN-only PE при помощи добавления extcomm RT vrf-x-vip.
                route-filter 10.50.50.50/32 exact;
            }
            then {
                community add vrf-x-vip;
                community add vrf-x-dci;
                community add vrf-x;
                accept;
            }
        }
        term dci {
            from {
                # Для сохранения оптимального роутинга между DC GW и разными VRF.
                route-filter 10.50.50.0/24 prefix-length-range /32-/32;
            }
            then {
                community add vrf-x-dci;
                accept;
            }
        }
        term master {
            from {
                # Fall-back.
                route-filter 10.50.50.0/24 exact;
            }
            then {
                community add vrf-x;
                accept;
            }
        }
    }
    policy-statement vrf-x_import {
        term other {
            from community [ vrf-x-dci vrf-x ];
            then accept;
        }
    }
    community vrf-x members target:1:1;
    community vrf-x-vip members target:1:2;
    community vrf-x-dci members target:1:3;
}
routing-instances {
    evpn-vlan-3550 {
        apply-groups ri-evpn;
        vlan-id 3550;
        interface ae0.3550;
        routing-interface irb.3550;
        # Формат RD RID:VPN для уникальности чтобы работал multipath. Уникальный в пределах системы.
        route-distinguisher 1.1.1.1:63550;
        vrf-target target:1:600003550;
    }
    vrf-x {
        apply-groups ri-vrf;
        interface irb.3550;
        # Формат RD RID:VPN для уникальности чтобы работал multipath. Уникальный в пределах системы.
        route-distinguisher 1.1.1.1:3550;
        vrf-import vrf-x_import;
        vrf-export vrf-x_export;
    }
}
routing-options {
    aggregate {
        defaults {
            discard;
        }
    }
    forwarding-table {
        # Для балансировки per-flow (не пугайтесь названия per-packet, это наследие).
        export Load_balance;
        # Необходим для EVPN, улучшает сходимость.
        dynamic-list-next-hop;
        # Уменьшает потерю трафика при выходе из работы ECMP/LAG одного из NH.
        ecmp-fast-reroute;
        # На новых версиях Junos OS по умолчанию включено только для EVPN.
        chained-composite-next-hop {
            ingress {
                evpn;
                l3vpn;
            }
        }
    }
}
protocols {
    bgp {
        # Позволяет назначить приоритеты анонсов для определенных типов маршрутов.
        export evpn-rt-priority-policy;
        # Необходимо выключить для улучшения сходимости и ускорения работы механизма core isolation.
        graceful-restart {
            disable;
        }
        multipath;
        group internal {
            type internal;
            local-address 1.1.1.1;
            family inet-vpn {
                unicast {
                    output-queue-priority priority 3;
                    route-refresh-priority priority 3;
                }
            }
            family evpn {
                signaling {
                    output-queue-priority priority 11;
                    route-refresh-priority priority 11;
                }
            }
            family route-target {
                output-queue-priority priority 16;
                route-refresh-priority priority 16;
            }
            peer-as 1;
            local-as 1;
            neighbor 2.2.2.1;
            neighbor 2.2.2.2;
        }
        # Позволяет выбрать. Low потенциально уменьшает загрузку ЦП при построении multipath.
        multipath-build-priority {
            low;
        }
    }
}
```

В данную конфигурацию не был включен VMTO в связи с отсутствием необходимости. Включается он отдельной командой в пределах EVPN RI и не вызывает отказа:
````
routing-instances {
    evpn-vlan-3550 {
        protocols {
            evpn {
                remote-ip-host-routes {
                    mpls-use-cnh;
                }
            }
        }
    }
}
```

## Ссылки
<b id="f1">1</b>. BGP MPLS-Based Ethernet VPN: [https://tools.ietf.org/html/rfc7432](https://tools.ietf.org/html/rfc7432) [↩](#a1)<br/>
<b id="f2">2</b>. EVPN Multihoming Overview: [https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html) [↩](#a2)<br/>
<b id="f3">3</b>. Understanding Automatically Generated ESIs in EVPN Networks: [https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-auto-esi.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-auto-esi.html) [↩](#a3)<br/>
<b id="f4">4</b>. Configuring Dynamic List Next Hop: [https://www.juniper.net/documentation/en_US/junos/topics/concept/configuring-dynamic-list.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/configuring-dynamic-list.html) [↩](#a4)<br/>
<b id="f5">5</b>. Understanding BGP Route Prioritization: [https://www.juniper.net/documentation/en_US/junos/topics/concept/bgp-route-prioritization-overview.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/bgp-route-prioritization-overview.html) [↩](#a5)<br/>
<b id="f6">6</b>. Сети для самых матёрых. Микровыпуск №7. EVPN: [https://habr.com/ru/post/316792/](https://habr.com/ru/post/316792/) [↩](#a6)<br/>
<b id="f7">7</b>. Proxy IP->MAC Advertisement in EVPNs: [https://tools.ietf.org/html/draft-rbickhart-evpn-ip-mac-proxy-adv-00](https://tools.ietf.org/html/draft-rbickhart-evpn-ip-mac-proxy-adv-00) [↩](#a7)<br/>
<b id="f8">8</b>. Ingress Virtual Machine Traffic Optimization: [https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-ingress-vmto.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-ingress-vmto.html) [↩](#a8)<br/>
<b id="f9">9</b>. EVPN Route Types: [http://bgphelp.com/2017/05/22/evpn-route-types/](http://bgphelp.com/2017/05/22/evpn-route-types/) [↩](#a9)<br/>
<b id="f10">10</b>. Multiprotocol Extensions for BGP-4: [https://tools.ietf.org/html/rfc4760](https://tools.ietf.org/html/rfc4760) [↩](#a10)<br/>
<b id="f11">11</b>. Composite next hop: [https://habr.com/ru/post/324268](https://habr.com/ru/post/324268) [↩](#a11)<br/>