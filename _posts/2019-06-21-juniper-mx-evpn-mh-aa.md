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
chassis {
    aggregated-devices {
        ethernet {
            device-count 1;
        }
    }
    network-services enhanced-ip; ## MUST
}
interfaces {
    xe-0/1/0 {
        description -to-P-router-MX480-6-;
        unit 0 {
            family inet {
                address 11.2.0.1/30;
            }
        }
    }
    xe-0/1/1 {
        apply-groups-except isis-mpls;
        description -to-CE-EX9200-1-;
        ether-options {
            802.3ad ae0;
        }
    }
    ae0 {
        description -LAG-to-CE-;
        flexible-vlan-tagging;
        encapsulation flexible-ethernet-services;
        esi {
            00:00:00:00:00:00:00:00:11:11; ## MUST manual (method 3) or can be auto-derive from lacp system-id
            all-active;
        }
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 00:00:00:1a:c0:02; ## MUST be and MUST be same on BOTH (or more) PE per ethernet segment
            }
        }
        unit 200 {
            encapsulation vlan-bridge;
            vlan-id 200;
        }
        unit 201 {
            encapsulation vlan-bridge;
            vlan-id 201;
        }
    }
    irb {
        unit 200 {
            family inet {
                address 200.0.1.1/16;
            }
            mac 00:00:02:00:01:01;
        }
        unit 201 {
            family inet {
                address 201.0.1.1/16;
            }
            mac 00:00:02:01:01:01;
        }
    }
    lo0 {
        unit 0 {
            family inet {
                address 1.1.1.1/32;
            }
        }
    }
}
forwarding-options {
    hash-key {
        family multiservice {
            payload {
                ip {
                    layer-3;
                }
            }
        }
    }
}
routing-options {
    router-id 1.1.1.1;
    autonomous-system 100;
    forwarding-table {
        export lbpp; ## MUST
        dynamic-list-next-hop; ## HIGHLY RECOMMENDED for better convergence time
    }
}
protocols {
    rsvp {
        interface xe-0/1/0.0 {
            link-protection; ## HIGHLY RECOMMENDED for better convergence time
        }
    }
    mpls {
        label-switched-path lsp-R1-1-to-R1-2 {
            from 1.1.1.1;
            to 1.1.2.2;
        }
		label-switched-path lsp-R1-1-to-R3 {
            from 1.1.1.1;
            to 3.3.3.3;
        }
        interface xe-0/1/0.0;
    }
    bgp {
        group Internal {
            type internal;
            local-address 1.1.1.1;
            family inet-vpn {
                unicast;
            }
            family evpn {
                signaling;
            }
            family route-target; ## RECOMMENDED for better memory utilization
            multipath; ## MUST in A/A scheme
            neighbor 2.2.2.2;
        }
    }
    ospf {
        traffic-engineering;
        area 0.0.0.0 {
            interface xe-0/1/0.0 {
                interface-type p2p;
            }
            interface lo0.0 {
                passive;
            }
        }
    }
    ldp {
        interface xe-0/1/0.0 {
            link-protection; ## HIGHLY RECOMMENDED for better convergence time
        }
        session-protection; ## HIGHLY RECOMMENDED for better convergence time
    }
    lldp {
        interface all;
        interface fxp0 {
            disable;
        }
    }
}
policy-options {
    policy-statement IP-VPN-ADD-COMM {
        term Balancer-host-route {
            from {
                route-filter 200.0.1.10/32 exact;
            }
            then {
                community add COMM-L3VPN-Clients;
                community add COMM-EVPN-Site-1;
                community add COMM-IPVPN-EVPN;
                accept;
            }
        }
        term EVPN-routes {
            from {
                route-filter 200.0.0.0/16 prefix-length-range /32-/32;
                route-filter 201.0.0.0/16 prefix-length-range /32-/32;
            }
            then {
                community add COMM-EVPN-Site-1;
                community add COMM-IPVPN-EVPN;
                accept;
            }
        }
        term All-other {
            then accept;
        }
    }
    policy-statement IP-VPN-DISCARD-EVPN {
        term Discard-EVPN-Redundand {
            from community COMM-EVPN-Site-1;
            then reject;
        }
        term Accept-IpVPN-from-Site-3 {
            from community COMM-IPVPN-EVPN;
            then accept;
        }
        term Accept-L3VPN-Clients {
            from community COMM-L3VPN-Clients;
            then accept;
        }
    }
    policy-statement lbpp {
        then {
            load-balance per-packet;
        }
    }
    community COMM-EVPN-Site-1 members target:2:1234;
    community COMM-IPVPN-EVPN members target:2:400;
    community COMM-L3VPN-Clients members target:3:300;
}
routing-instances {
    EVPN_vlan_200 {
        instance-type evpn;
        vlan-id 200;
        interface ae0.200;
        routing-interface irb.200;
        route-distinguisher 1.1.1.1:200;
        vrf-target target:2:200;
        protocols {
            evpn {
                default-gateway do-not-advertise; ## MUST for distributed IRB and anycast GW
            }
        }
    }
    EVPN_vlan_201 {
        instance-type evpn;
        vlan-id 201;
        interface ae0.201;
        routing-interface irb.201;
        route-distinguisher 1.1.1.1:201;
        vrf-target target:2:201;
        protocols {
            evpn;
        }
    }
    L3VPN_for_EVPN {
        instance-type vrf;
        interface irb.200;
        interface irb.201;
        route-distinguisher 1.1.1.1:300;
        vrf-import IP-VPN-DISCARD-EVPN;
        vrf-export IP-VPN-ADD-COMM;
        vrf-table-label;
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
