---
layout: post
title: "Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX"
tags: juniper mpls bgp evpn l3vpn
author: "somovis"
---

Самое необходимое про EVPN MH A/A + L3VPN на Juniper MX

## Коротко о EVPN

EVPN использует новое BGP NLRI именуемое EVPN NLRI. Это NLRI передаётся в существующем AFI 25 \(L2VPN\) и новом SAFI 70 \(EVPN\).

Проблемы VPLS или что было до EVPN:

- В сценарии multihoming \(далее MH\) возможен только active/standby
- Все виды VPLS не предоставляют расширенных возможностей L3, за исключением банального добавления BVI/IRB интерфейса в VPLS домен для inter vlan routing, в следствии чего, на оборудовании \(зависит от реализации\) выполняется больше операций lookup и возможен не оптимальный packet flow
- MAC-адреса изучаются исключительно на уровне data plane, что приводит к увеличению флуда BUM трафика в сети
- re-learn и частичная потеря трафика во время VMotion

Решения проблем, которые предлагает EVPN:

- В сценарии MH для каждого ethernet segment \(далее ES\) возможно использовать:
  - active/standby \(a/s\)
  - active/active \(a/a\)
- У EVPN нет разных типов, что облегчает понимание технологии, поиск и устранение неисправностей, а так же EVPN просто расширяется за счёт добавления новых [route types](http://bgphelp.com/2017/05/22/evpn-route-types/), но, при этом, не все route types и возможности EVPN поддерживаются разными производителями и/или аппаратными/программными платформами
- EVPN предоставляет расширенные возможности для inter vlan routing, имеет оптимальный packet flow, большой набор возможностей для DCI
- MAC-адреса изучаются на уровне control plane \(представьте себе поведение L3VPN, только не по отношению к IP-адресам, а к MAC-адресам\), это позволяет получить оптимальный packet flow и уменьшить BUM трафик в сети
- В 18.4 появилась поддержка [VMTO](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-ingress-vmto.html), что позволяет оптимизировать входящий трафик на шлюз по умолчанию во время VMotion

## Чем хорош EVPN MH, почему именно A/A и причём тут L3VPN

Чтобы не быть переводчиком, рекомендую прочитать [RFC](https://tools.ietf.org/html/rfc7432) и [реализацию Juniper](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html), а русскоязычным пользователям для более детального ознакомления [статью на habr](https://habr.com/ru/post/316792/).

Краткое описание:

- EVPN MH полезен тем, что можно использовать более одного PE маршрутизатора за каждый ES для обеспечения резервирования ES
- EVPN MH A/A полезен тем, что каждый PE маршрутизатор за каждый ES способен отправлять и принимать трафик, это позволит использовать load sharing по всем интерфейсам всех PE-CE, дополнительно получаем "бонус" в виде усложнения поиска и устранения неисправностей, но мы этого не боимся
- L3VPN необходим для того, чтобы обеспечить связность между L3VPN-only маршрутизаторами и маршрутизаторами с поддержкой EVPN + L3VPN

## А теперь соберем всё вместе

До 18.4 при использовании EVPN MH A/A + L3VPN:
- EVPN route type 2 MAC+IP анонсировались со всех PE за ES, но, при этом, в L3VPN host prefix \(на Juniper по умолчанию при добавлении интерфейса IRB в EVPN RI и VRF RI импортируются префиксы из EVPN route type 2 MAC+IP в VRF как /32 префиксы\) анонсировались только с того PE маршрутизатора, который был выбран DF \(EVPN route type 2 MAC+IP изучаются в напрямую подключенном ES при получении ARP\), что в свою очередь не позволяло сделать честный A/A для всего трафика за этот ES

Начиная с 18.4 при использовании EVPN MH A/A + L3VPN:
- В 18.4 добавили [Multihomed Proxy MAC and IP Address Route Advertisement](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-bgp-multihoming-overview.html#jd0e651), что в совокупности с VMTO и нехитрой политикой позволило анонсировать EVPN route type 2 MAC+IP и L3VPN host prefix со всех PE маршрутизаторов той площадки, на которой были изучены EVPN route type 2 MAC+IP в напрямую подключенном ES

Данный сценарий был проверен с успешным результатом на оборудовании Juniper MX204 18.4R1-S1.

## Топология

![Topology](/images/juniper-mx-evpn-mh-aa-topology.png)

## Пример конфигурации

Конфигурация может изменятся по мере реализации новых возможностей производителем.

_Меня попросили для ясности описать почему использовались те или иные команды в конфигурации. Я опишу в комментариях к конфигурации часть важных на мой взгляд команд. Для всего остального рекомендую воспользоваться [CLI Explorer](https://apps.juniper.net/cli-explorer/)_

```
groups {
    interface-evpn-all-active { ## for compressed configuration view
        interfaces {
            <*> {
                unit <*> {
                    encapsulation vlan-bridge;
                    esi { ## vESI - MUST for better user experience and manual interface statement
                        auto-derive {
                            lacp; ## autoderive ESI from LACP system-id
                        }
                        all-active;
                    }
                }
            }
        }
    }
}
system {
    host-name pe1;
    no-redirects;
    arp { ## host routes depends on ARP, choose your setup
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
    network-services enhanced-ip; ## MUST, system restart required
}
interfaces {
    ge-0/0/3 {
        ether-options {
            802.3ad ae0;
        }
    }
    ae0 {
        flexible-vlan-tagging;
        encapsulation flexible-ethernet-services;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 0a:00:00:01:01:01; ## MUST be and MUST be same on BOTH (or more) PE per ethernet segment
            }
        }
        unit 500 {
            apply-groups interface-evpn-all-active;
            vlan-id 500;
        }
    }
    irb {
        unit 500 {
            family inet {
                address 10.10.10.254/24;
            }
        }
    }
    lo0 {
        unit 0 {
            description "GRT";
            family inet {
                address 10.100.0.1/32;
            }
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
            payload {
                ip {
                    layer-3;
                }
            }
        }
    }
    enhanced-hash-key {
        family inet {
            incoming-interface-index;
        }
        family mpls {
            incoming-interface-index;
        }
        family multiservice {
            incoming-interface-index;
        }
    }
}
policy-options {
    policy-statement pfe_lbl {
        term 1 {
            then {
                load-balance per-packet;
            }
        }
    }
    policy-statement vrfx_export {
        term vip-000 {
            from {
                route-filter 10.10.10.1/32 exact;
                route-filter 10.10.10.2/32 exact;
            }
            then {
                community add vrfx-vip-000; ## for legacy load balancers and his vip address for export to l3vpn-only clients
                community add vrfx;
                accept;
            }
        }
        term other {
            then {
                community add vrfx;
                accept;
            }
        }
    }
    policy-statement vrfx_import {
        term vip-000 {
            from community vrfx-vip-000;
            then accept;
        }
        term other {
            from community vrfx;
            then accept;
        }
    }
    community vrfx members target:65000:100;
    community vrfx-vip-000 members target:65000:100030000;
}
routing-instances {
    EVPN_vlan_500 {
        instance-type evpn;
        vlan-id 500;
        interface ae0.500;
        routing-interface irb.500;
        route-distinguisher 10.100.0.1:65500;
        vrf-target target:37283:601010500;
        protocols {
            evpn {
                default-gateway do-not-advertise; ## MUST for distributed IRB and anycast GW
            }
        }
    }
    L3VPN_for_EVPN_vlan_500 {
        instance-type vrf;
        interface irb.500;
        route-distinguisher 10.100.0.1:500;
        vrf-import vrfx_import;
        vrf-export vrfx_export;
        vrf-table-label;
    }
}
routing-options {
    router-id 10.100.0.1;
    autonomous-system 65000;
    forwarding-table {
        export pfe_lbl; ## MUST
        dynamic-list-next-hop; ## HIGHLY RECOMMENDED for better convergence time
        ecmp-fast-reroute; ## HIGHLY RECOMMENDED for better convergence time
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
        multipath; ## MUST in A/A scheme
        group internal {
            type internal;
            local-address 10.100.0.1;
            family inet-vpn {
                unicast;
            }
            family evpn {
                signaling;
            }
            family route-target; ## RECOMMENDED for better memory utilization
            peer-as 65000;
            local-as 65000;
            neighbor 10.100.0.101;
            neighbor 10.100.0.102;
        }
        multipath-build-priority {
            low;
        }
    }
    evpn {
        remote-ip-host-routes; ## HIGHLY RECOMMENDED for better convergence time
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
