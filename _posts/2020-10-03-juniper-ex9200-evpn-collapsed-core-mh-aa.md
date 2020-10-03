---
layout: post
title: "Коротко про EVPN/MPLS Collapsed Core MH A/A и L3VPN в кампусе на базе Juniper EX9200 + базовая конфигурация"
tags: juniper mpls bgp evpn l3vpn
author: "somovis"
---

В данной заметке я хочу поделится опытом построения сети передачи данных кампуса на базе оборудования **Juniper EX9200** используя **EVPN Collapsed Core**, более детально можно ознакомится в презентации по ссылке - [Evolved Campus Core:
An EVPN Framework for Campus Networks](https://www.juniper.net/assets/us/en/local/pdf/nxtwork/network-vpn-test-without-dhcp-relay-evolved-campus-core.pdf)

## Вступление

 Описание работы механизма EVPN пропущено, так как данная заметка практически полностью копирует заметку написанную ранее, рекомендую ознакомится с ней по ссылке - [Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX
](https://ntwrk.today/2019/06/21/juniper-mx-evpn-mh-aa.html)
Вы можете спросить: "зачем же писать еще одну заметку?".
Ответ будет простым: **в данной заметке речь пойдет именно про особенности сети передачи данных в кампусе**, что может быть полезно для инженеров с упором на корпоративных заказчиков.

От себя я хочу добавить отличие в платформе с MX и разницу в конфигурации:
1. На EX9200 не поддерживаются следующие опции:
   1. ```set chassis ecmp-alb;```
   2. ```set chassis network-services enhanced-ip```;
   3. ```set routing-options forwarding-table ecmp-fast-reroute```.
2. В силу того, что в кампусе не нужна такая гранулярность, как она была нужна в ДЦ во время миграции между устройствами разных производителей с сохранением сервиса, мы упрощаем конфигурацию в следующем:
   1. Вместо VLAN-based используем VLAN-Aware bundle services;
   2. Вместо ESI per IFL используем ESI per IFD.

Многие так же могут заметить что используется dataplane MPLS для EVPN, а не VXLAN, причина кроется в следующем:
1. Уже есть MPLS backbone и многие сервисы работают именно поверх MPLS, поэтому проще и быстрее предоставить сервис для пользователей в кампусе используя MPLS;
2. На площадке всего 2 устройства способных работать с EVPN/VXLAN и/или EVPN/MPLS, поэтому, чтобы не делать склеивание dataplane VXLAN/MPLS на этих самых устройствах, проще использовать только MPLS;
3. Исходя из предыдущего пункта, мы так же снижаем вероятность ошибки в ПО используя меньшее количество протоколов.

## Топология

![Topology](/images/2020-10-03-juniper-ex9200-evpn-collapsed-core-mh-aa-topology.png)

## Плюсы и минусы в сравнении с MC-LAG

Я не могу пропустить это сравнение, так как оно все же может быть полезным.

**Плюсы**:
1. Отсутствие необходимости в дополнительном протоколе резервирования Ethernet - MC-LAG/Virtual-Chassis или прочих;
2. Отсутствие необходимости в дополнительном протоколе резервирования шлюза - VRRP или прочих;
3. Возможность использовать конфигурацию Active-Active как для L2, так и для L3 трафика;
4. Возможность горизонтального масштабирования (более двух устройств), но это не кейс для кампуса, однако все равно плюс;
5. В случае необходимости нативными средствами EVPN (если другая сторона поддерживает EVPN/MPLS) возможно растянуть L2 домен;
6. Сокращается количество запущенных протоколов на устройстве, в следствии чего уеньшается вероятность перебоя в работе устройства из-за возможной ошибки в одном из протоколов.

**Минсусы**:
1. Невозможно использовать Active-Standby для L2 и для L3 трафика одновременно. Да, можно поменять Local-preference для L3 и сделать EVPN DF preference-based, но не все можно настроить руками и для этого нужен дополнительный внешний механизм - скрипты on-box/eem и прочее;
2. Новый протокол, еще недостаточно проверен временем и в реализации разных вендоров до сих пор встречаются мажорные баги;
3. В дополнение к предыдущему пункту - не все инженеры могут быть готовы к поиску и исправлению проблем в сети передачи данных на базе EVPN; 
4. Невозможно обеспечить ZTP для нижестоящих устройств в некоторых случаях без предварительного конфигурирования AC;
5. Невозможно использовать динамическую маршрутизацию на IRB с anycast-gateway.

>Если что-то вдруг забыл указать, напишите в ЛС, добавлю.

## Пример конфигурации

> **Конфигурация может изменятся по мере реализации новых возможностей производителем**.

> **Особенности:** Конфигурация тестировалась на Junos 19.3R3, так как **начиная с Junos 19.3R1 добавили возможность использовать DHCP-Relay в EVPN/MPLS**.

> **Примечание:** Часть важных опций прокомментирована и конфигурация, не имеющая отношение к делу, удалена.

**AR01**:
```
version 19.3R3.6;
groups {
    interface-ospf {
        protocols {
            ospf {
                area 0.0.0.0 {
                    interface <*> {
                        interface-type p2p;
                        node-link-protection;
                        flood-reduction;
                        bfd-liveness-detection {
                            version automatic;
                            minimum-interval 300;
                            multiplier 3;
                        }
                        ldp-synchronization;
                    }
                }
            }
        }
    }
    interface-mpls {
        interfaces {
            <*> {
                unit 0 {
                    family mpls {
                        maximum-labels 5;
                    }
                }
            }
        }
    }
    ri-vrf {
        routing-instances {
            <*> {
                instance-type vrf;
                routing-options {
                    multipath;
                    protect core;
                    auto-export;
                }
                vrf-table-label;
            }
        }
    }
    dhcp-relay-group {
        routing-instances {
            <*> {
                forwarding-options {
                    dhcp-relay {
                        overrides {
                            trust-option-82;
                        }
                        relay-option-82 {
                            circuit-id {
                                prefix {
                                    routing-instance-name;
                                }
                                no-vlan-interface-name;
                            }
                        }
                        forward-only;
                        forward-only-replies;
                        server-group {
                            dhcp-servers {
                                10.10.10.10;
                                10.10.20.20;
                            }
                        }
                        active-server-group dhcp-servers;
                    }
                }
            }
        }
    }
    interface-evpn-all-active {
        interfaces {
            <*> {
                flexible-vlan-tagging;
                mtu 9192;
                encapsulation flexible-ethernet-services;
                esi {
                    auto-derive {
                        lacp;
                    }
                    all-active;
                }
                unit 0 {
                    encapsulation vlan-bridge;
                    family ethernet-switching {
                        interface-mode trunk;
                        vlan {
                            members [ VLAN5 VLAN401 ];
                        }
                    }
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
                        remote-ip-host-routes;
                        label-allocation per-instance;
                        default-gateway do-not-advertise;
                        duplicate-mac-detection {
                            detection-threshold 5;
                            detection-window 180;
                            auto-recovery-time 5;
                        }
                    }
                }
            }
        }
    }
system {
    host-name ar01;
    commit {
        persist-groups-inheritance;
        delta-export;
    }
    no-redirects;
    arp {
        aging-timer 5;
        passive-learning;
        purging;
        gratuitous-arp-on-ifup;
        gratuitous-arp-delay 3;
    }
    internet-options {
        path-mtu-discovery;
        tcp-drop-synfin-set;
        no-tcp-reset drop-all-tcp;
    }
    compress-configuration-files;
    max-configurations-on-flash 49;
    archival {
        configuration {
            transfer-on-commit;
            archive-sites {
                ftp://***/upload/configurations/;
            }
        }
    }
    license {
        autoupdate {
            url https://ae1.juniper.net/junos/key_retrieval;
        }
    }
    ddos-protection {
        global {
            flow-detection;
        }
        protocols {
            dhcpv4 {
                aggregate {
                    flow-level-detection { ## Warning: 'flow-level-detection' is deprecated
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
                discover {
                    flow-level-detection {
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
                bad-packets {
                    flow-level-detection {
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
            }
        }
    }
    ntp {
        peer 10.100.0.97;
        server 10.50.77.65 prefer;
        server 10.50.67.65 prefer;
    }
}
chassis {
    craft-lockout;
    aggregated-devices {
        ethernet {
            device-count 24;
        }
    }
    alarm {
        management-ethernet {
            link-down ignore;
        }
    }
}
interfaces {
    xe-0/0/0 {
        hold-time up 120000 down 1;
        gigether-options {
            802.3ad ae0;
        }
    }
    xe-0/0/1 {
        hold-time up 120000 down 1;
        gigether-options {
            802.3ad ae1;
        }
    }
    xe-0/3/5 {
        apply-groups interface-mpls;
        description "link with p1";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.192/31;
            }
        }
    }
    xe-0/3/6 {
        apply-groups interface-mpls;
        description "link with ar02";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.196/31;
            }
        }
    }
    xe-0/3/7 {
        apply-groups interface-mpls;
        description "link with ar02";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.198/31;
            }
        }
    }
    ae0 {
        apply-groups interface-evpn-all-active;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 0a:50:00:00:00:00;
            }
        }
    }
    ae1 {
        apply-groups interface-evpn-all-active;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 0a:50:00:00:00:01;
            }
        }
    }
    irb {
        arp-l2-validate;
        unit 5 {
            description vpn-test-without-dhcp-relay;
            family inet {
                address 10.100.27.254/24;
            }
            mac 0a:50:00:00:00:05;
        }
        unit 401 {
            description vpn-test-with-dhcp-relay;
            family inet {
                address 10.4.30.254/24;
            }
            mac 0a:50:00:00:04:01;
        }
    }
    lo0 {
        unit 0 {
            description "GRT OSPF";
            family inet {
                filter {
                    input-list [ discard-frags accept-bfd accept-iccp accept-igp accept-ldp-rsvp accept-bgp accept-vrrp accept-telnet accept-ftp accept-remote-auth accept-common-services accept-established discard-all ];
                }
                address 10.100.0.96/32;
            }
        }
    }
}
forwarding-options {
    storm-control-profiles default {
        all {
            bandwidth-percentage 1;
            no-registered-multicast;
        }
    }
    load-balance {
        per-flow {
            hash-seed;
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
    policy-statement vpn-test-without-dhcp-relay_export {
        from protocol direct;
        then {
            community add vpn-test-without-dhcp-relay;
            accept;
        }
    }
    policy-statement vpn-test-without-dhcp-relay_import {
        from community vpn-test-without-dhcp-relay;
        then accept;
    }
    policy-statement vpn-test-with-dhcp-relay_export {
        from protocol direct;
        then {
            community add vpn-test-with-dhcp-relay;
            accept;
        }
    }
    policy-statement vpn-test-with-dhcp-relay_import {
        from community vpn-test-with-dhcp-relay;
        then accept;
    }
    community vpn-test-without-dhcp-relay members target:65535:65535;
    community vpn-test-with-dhcp-relay members target:65534:65534;
}
routing-instances {
    evpn-collapsed-core {
        instance-type virtual-switch;
        protocols {
            evpn {
                remote-ip-host-routes;
                label-allocation per-instance;
                extended-vlan-list [ 5 401 ];
                default-gateway do-not-advertise;
                duplicate-mac-detection {
                    detection-threshold 5;
                    detection-window 180;
                    auto-recovery-time 5;
                }
            }
        }
        interface ae0.0;
        interface ae1.0;
        switch-options {
            mac-table-size {
                10240;
            }
            mac-ip-table-size {
                10240;
            }
            interface-mac-limit {
                10240;
            }
            interface-mac-ip-limit {
                10240;
            }
        }
        vrf-target target:65535:650000000;
        vlans {
            VLAN401 {
                vlan-id 401;
                l3-interface irb.401;
            }
            VLAN5 {
                vlan-id 5;
                l3-interface irb.5;
            }
        }
    }
    vpn-test-without-dhcp-relay {
        apply-groups ri-vrf;
        interface irb.5;
        vrf-import vpn-test-without-dhcp-relay_import;
        vrf-export vpn-test-without-dhcp-relay_export;
    }
    vpn-test-with-dhcp-relay {
        apply-groups [ ri-vrf dhcp-relay-group ];
        interface irb.401;
        forwarding-options {
            dhcp-relay {
                group vpn-test-with-dhcp-relay {
                    interface irb.401;
                }
            }
        }
        vrf-import vpn-test-with-dhcp-relay_import;
        vrf-export vpn-test-with-dhcp-relay_export;
    }
}
routing-options {
    route-distinguisher-id 10.100.0.96;
    forwarding-table {
        export Load_balance;
        dynamic-list-next-hop;
        chained-composite-next-hop {
            ingress {
                evpn;
                l3vpn;
            }
        }
    }
    router-id 10.100.0.96;
    autonomous-system 65535
    aggregate {
        defaults {
            discard;
        }
    }
    protect core;
}
protocols {
    ospf {
        backup-spf-options {
            remote-backup-calculation;
            per-prefix-calculation all;
            node-link-degradation;
        }
        traffic-engineering;
        area 0.0.0.0 {
            interface xe-0/3/5.0 {
                apply-groups interface-ospf;
            }
            interface xe-0/3/6.0 {
                apply-groups interface-ospf;
            }
            interface xe-0/3/7.0 {
                apply-groups interface-ospf;
            }
            interface lo0.0 {
                passive;
            }
            interface fxp0.0 {
                disable;
            }
        }
        spf-options {
            delay 50;
            holddown 5000;
            rapid-runs 3;
        }
        reference-bandwidth 100g;
        lsa-refresh-interval 30;
        no-rfc-1583;
    }
    evpn {
        no-core-isolation; #Одна из самых важных опций для EVPN Collapsed Core - так как у нас BGP сессия overlay EVPN построена между двумя напрямую соединенными устройствами, если вдруг одно из утсройств будет недоступно по тем или иным причиным, порвется BGP сессия, а нам не нужно чтобы клиентские интерфейсы в случае падения BGP сессии тоже упали.
    }
    bgp {
        vpn-apply-export;
        group internal {
            type internal;
            local-address 10.100.0.96;
            family inet-vpn {
                unicast {
                    output-queue-priority priority 3;
                    route-refresh-priority priority 3;
                }
            }
            family route-target {
                output-queue-priority priority 16;
                route-refresh-priority priority 16;
            }
            peer-as 65535;
            local-as 65535;
            neighbor 10.100.0.243;
            neighbor 10.100.0.244;
        }
        group evpn-collapsed-core { ## а вот и сама группа пиринга с соседним устройством в кампусе.
            type internal;
            local-address 10.100.0.96;
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
            peer-as 65535;
            local-as 65535;
            bfd-liveness-detection {
                minimum-interval 1000;
                multiplier 3;
                session-mode automatic;
            }
            neighbor 10.100.0.97;
        }
        local-preference 350;
        mtu-discovery;
        log-updown;
        damping;
        export evpn-rt-priority-policy;
        graceful-restart {
            disable;
        }
        multipath;
        multipath-build-priority {
            low;
        }
    }
    ldp {
        auto-targeted-session;
        track-igp-metric;
        mtu-discovery;
        deaggregate;
        interface xe-0/3/5.0 {
            link-protection;
        }
        interface xe-0/3/6.0 {
            link-protection;
        }
        interface xe-0/3/7.0 {
            link-protection;
        }
        interface fxp0.0 {
            disable;
        }
        interface lo0.0;
        session-protection;
    }
    mpls {
        interface xe-0/3/5.0;
        interface xe-0/3/6.0;
        interface xe-0/3/7.0;
        interface fxp0.0 {
            disable;
        }
    }
    lldp {
        port-id-subtype interface-name;
        interface all;
    }
    lldp-med {
        interface all;
    }
}
```

**AR02**:
```
version 19.3R3.6;
groups {
    interface-ospf {
        protocols {
            ospf {
                area 0.0.0.0 {
                    interface <*> {
                        interface-type p2p;
                        node-link-protection;
                        flood-reduction;
                        bfd-liveness-detection {
                            version automatic;
                            minimum-interval 300;
                            multiplier 3;
                        }
                        ldp-synchronization;
                    }
                }
            }
        }
    }
    interface-mpls {
        interfaces {
            <*> {
                unit 0 {
                    family mpls {
                        maximum-labels 5;
                    }
                }
            }
        }
    }
    ri-vrf {
        routing-instances {
            <*> {
                instance-type vrf;
                routing-options {
                    multipath;
                    protect core;
                    auto-export;
                }
                vrf-table-label;
            }
        }
    }
    dhcp-relay-group {
        routing-instances {
            <*> {
                forwarding-options {
                    dhcp-relay {
                        overrides {
                            trust-option-82;
                        }
                        relay-option-82 {
                            circuit-id {
                                prefix {
                                    routing-instance-name;
                                }
                                no-vlan-interface-name;
                            }
                        }
                        forward-only;
                        forward-only-replies;
                        server-group {
                            dhcp-servers {
                                10.10.10.10;
                                10.10.20.20;
                            }
                        }
                        active-server-group dhcp-servers;
                    }
                }
            }
        }
    }
    interface-evpn-all-active {
        interfaces {
            <*> {
                flexible-vlan-tagging;
                mtu 9192;
                encapsulation flexible-ethernet-services;
                esi {
                    auto-derive {
                        lacp;
                    }
                    all-active;
                }
                unit 0 {
                    encapsulation vlan-bridge;
                    family ethernet-switching {
                        interface-mode trunk;
                        vlan {
                            members [ VLAN5 VLAN401 ];
                        }
                    }
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
                        remote-ip-host-routes;
                        label-allocation per-instance;
                        default-gateway do-not-advertise;
                        duplicate-mac-detection {
                            detection-threshold 5;
                            detection-window 180;
                            auto-recovery-time 5;
                        }
                    }
                }
            }
        }
    }
system {
    host-name ar02;
    commit {
        persist-groups-inheritance;
        delta-export;
    }
    no-redirects;
    arp {
        aging-timer 5;
        passive-learning;
        purging;
        gratuitous-arp-on-ifup;
        gratuitous-arp-delay 3;
    }
    internet-options {
        path-mtu-discovery;
        tcp-drop-synfin-set;
        no-tcp-reset drop-all-tcp;
    }
    compress-configuration-files;
    max-configurations-on-flash 49;
    archival {
        configuration {
            transfer-on-commit;
            archive-sites {
                ftp://***/upload/configurations/;
            }
        }
    }
    license {
        autoupdate {
            url https://ae1.juniper.net/junos/key_retrieval;
        }
    }
    ddos-protection {
        global {
            flow-detection;
        }
        protocols {
            dhcpv4 {
                aggregate {
                    flow-level-detection { ## Warning: 'flow-level-detection' is deprecated
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
                discover {
                    flow-level-detection {
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
                bad-packets {
                    flow-level-detection {
                        subscriber on;
                        logical-interface on;
                        physical-interface on;
                    }
                }
            }
        }
    }
    ntp {
        peer 10.100.0.97;
        server 10.50.77.65 prefer;
        server 10.50.67.65 prefer;
    }
}
chassis {
    craft-lockout;
    aggregated-devices {
        ethernet {
            device-count 24;
        }
    }
    alarm {
        management-ethernet {
            link-down ignore;
        }
    }
}
interfaces {
    xe-0/0/0 {
        hold-time up 120000 down 1;
        gigether-options {
            802.3ad ae0;
        }
    }
    xe-0/0/1 {
        hold-time up 120000 down 1;
        gigether-options {
            802.3ad ae1;
        }
    }
    xe-0/3/5 {
        apply-groups interface-mpls;
        description "link with p2";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.194/31;
            }
        }
    }
    xe-0/3/6 {
        apply-groups interface-mpls;
        description "link with ar01";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.197/31;
            }
        }
    }
    xe-0/3/7 {
        apply-groups interface-mpls;
        description "link with ar01";
        mtu 9192;
        unit 0 {
            family inet {
                address 10.100.4.199/31;
            }
        }
    }
    ae0 {
        apply-groups interface-evpn-all-active;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 0a:50:00:00:00:00;
            }
        }
    }
    ae1 {
        apply-groups interface-evpn-all-active;
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
                system-id 0a:50:00:00:00:01;
            }
        }
    }
    irb {
        arp-l2-validate;
        unit 5 {
            description vpn-test-without-dhcp-relay;
            family inet {
                address 10.100.27.254/24;
            }
            mac 0a:50:00:00:00:05;
        }
        unit 401 {
            description vpn-test-with-dhcp-relay;
            family inet {
                address 10.4.30.254/24;
            }
            mac 0a:50:00:00:04:01;
        }
    }
    lo0 {
        unit 0 {
            description "GRT OSPF";
            family inet {
                filter {
                    input-list [ discard-frags accept-bfd accept-iccp accept-igp accept-ldp-rsvp accept-bgp accept-vrrp accept-telnet accept-ftp accept-remote-auth accept-common-services accept-established discard-all ];
                }
                address 10.100.0.96/32;
            }
        }
    }
}
forwarding-options {
    storm-control-profiles default {
        all {
            bandwidth-percentage 1;
            no-registered-multicast;
        }
    }
    load-balance {
        per-flow {
            hash-seed;
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
    policy-statement vpn-test-without-dhcp-relay_export {
        from protocol direct;
        then {
            community add vpn-test-without-dhcp-relay;
            accept;
        }
    }
    policy-statement vpn-test-without-dhcp-relay_import {
        from community vpn-test-without-dhcp-relay;
        then accept;
    }
    policy-statement vpn-test-with-dhcp-relay_export {
        from protocol direct;
        then {
            community add vpn-test-with-dhcp-relay;
            accept;
        }
    }
    policy-statement vpn-test-with-dhcp-relay_import {
        from community vpn-test-with-dhcp-relay;
        then accept;
    }
    community vpn-test-without-dhcp-relay members target:65535:65535;
    community vpn-test-with-dhcp-relay members target:65534:65534;
}
routing-instances {
    evpn-collapsed-core {
        instance-type virtual-switch;
        protocols {
            evpn {
                remote-ip-host-routes;
                label-allocation per-instance;
                extended-vlan-list [ 5 401 ];
                default-gateway do-not-advertise;
                duplicate-mac-detection {
                    detection-threshold 5;
                    detection-window 180;
                    auto-recovery-time 5;
                }
            }
        }
        interface ae0.0;
        interface ae1.0;
        switch-options {
            mac-table-size {
                10240;
            }
            mac-ip-table-size {
                10240;
            }
            interface-mac-limit {
                10240;
            }
            interface-mac-ip-limit {
                10240;
            }
        }
        vrf-target target:65535:650000000;
        vlans {
            VLAN401 {
                vlan-id 401;
                l3-interface irb.401;
            }
            VLAN5 {
                vlan-id 5;
                l3-interface irb.5;
            }
        }
    }
    vpn-test-without-dhcp-relay {
        apply-groups ri-vrf;
        interface irb.5;
        vrf-import vpn-test-without-dhcp-relay_import;
        vrf-export vpn-test-without-dhcp-relay_export;
    }
    vpn-test-with-dhcp-relay {
        apply-groups [ ri-vrf dhcp-relay-group ];
        interface irb.401;
        forwarding-options {
            dhcp-relay {
                group vpn-test-with-dhcp-relay {
                    interface irb.401;
                }
            }
        }
        vrf-import vpn-test-with-dhcp-relay_import;
        vrf-export vpn-test-with-dhcp-relay_export;
    }
}
routing-options {
    route-distinguisher-id 10.100.0.96;
    forwarding-table {
        export Load_balance;
        dynamic-list-next-hop;
        chained-composite-next-hop {
            ingress {
                evpn;
                l3vpn;
            }
        }
    }
    router-id 10.100.0.96;
    autonomous-system 65535
    aggregate {
        defaults {
            discard;
        }
    }
    protect core;
}
protocols {
    ospf {
        backup-spf-options {
            remote-backup-calculation;
            per-prefix-calculation all;
            node-link-degradation;
        }
        traffic-engineering;
        area 0.0.0.0 {
            interface xe-0/3/5.0 {
                apply-groups interface-ospf;
            }
            interface xe-0/3/6.0 {
                apply-groups interface-ospf;
            }
            interface xe-0/3/7.0 {
                apply-groups interface-ospf;
            }
            interface lo0.0 {
                passive;
            }
            interface fxp0.0 {
                disable;
            }
        }
        spf-options {
            delay 50;
            holddown 5000;
            rapid-runs 3;
        }
        reference-bandwidth 100g;
        lsa-refresh-interval 30;
        no-rfc-1583;
    }
    evpn {
        no-core-isolation;
    }
    bgp {
        vpn-apply-export;
        group internal {
            type internal;
            local-address 10.100.0.96;
            family inet-vpn {
                unicast {
                    output-queue-priority priority 3;
                    route-refresh-priority priority 3;
                }
            }
            family route-target {
                output-queue-priority priority 16;
                route-refresh-priority priority 16;
            }
            peer-as 65535;
            local-as 65535;
            neighbor 10.100.0.243;
            neighbor 10.100.0.244;
        }
        group evpn-collapsed-core {
            type internal;
            local-address 10.100.0.96;
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
            peer-as 65535;
            local-as 65535;
            bfd-liveness-detection {
                minimum-interval 1000;
                multiplier 3;
                session-mode automatic;
            }
            neighbor 10.100.0.97;
        }
        local-preference 350;
        mtu-discovery;
        log-updown;
        damping;
        export evpn-rt-priority-policy;
        graceful-restart {
            disable;
        }
        multipath;
        multipath-build-priority {
            low;
        }
    }
    ldp {
        auto-targeted-session;
        track-igp-metric;
        mtu-discovery;
        deaggregate;
        interface xe-0/3/5.0 {
            link-protection;
        }
        interface xe-0/3/6.0 {
            link-protection;
        }
        interface xe-0/3/7.0 {
            link-protection;
        }
        interface fxp0.0 {
            disable;
        }
        interface lo0.0;
        session-protection;
    }
    mpls {
        interface xe-0/3/5.0;
        interface xe-0/3/6.0;
        interface xe-0/3/7.0;
        interface fxp0.0 {
            disable;
        }
    }
    lldp {
        port-id-subtype interface-name;
        interface all;
    }
    lldp-med {
        interface all;
    }
}
```


## Ссылки
<b id="f1">1</b>. [Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX](https://ntwrk.today/2019/06/21/juniper-mx-evpn-mh-aa.html) [↩](#a1)<br/>
<b id="f2">2</b>. [Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX](https://www.juniper.net/assets/us/en/local/pdf/nxtwork/network-vpn-test-without-dhcp-relay-evolved-campus-core.pdf) [↩](#a2)<br/>
<b id="f3">3</b>. [Understanding When to Disable EVPN-VXLAN Core Isolation](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-vxlan-core-isolation-disabling.html) [↩](#a3)<br/>
