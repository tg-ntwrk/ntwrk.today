---
layout: post
title: "Коротко про EVPN/MPLS Collapsed Core MH A/A и L3VPN в кампусе на базе Juniper EX9200 + базовая конфигурация"
tags: juniper mpls bgp evpn l3vpn
author: "somovis"
---

В данной заметке я хочу поделится опытом построения сети передачи данных кампуса на базе оборудования **Juniper EX9200** используя **EVPN Collapsed Core**, более детально можно ознакомится в презентации по ссылке - [Evolved Campus Core:
An EVPN Framework for Campus Networks](https://www.juniper.net/assets/us/en/local/pdf/nxtwork/network-infrastructure-evolved-campus-core.pdf)

## Вступление

Описание работы механизма EVPN пропущено, так как данная заметка практически полностью копирует заметку, написанную ранее, рекомендую ознакомится с ней по ссылке - [Самое необходимое про EVPN/MPLS MH A/A + L3VPN на Juniper MX
](https://ntwrk.today/2019/06/21/juniper-mx-evpn-mh-aa.html)

Отличие будет в том, что **в данной заметке речь пойдет именно про особенности сети передачи данных в кампусе**.

***От себя я хочу добавить отличие в платформе с MX и разницу в конфигурации:***
1. На EX9200 не поддерживаются следующие опции:
   1. ```set chassis ecmp-alb```;
   2. ```set chassis network-services enhanced-ip```;
   3. ```set routing-options forwarding-table ecmp-fast-reroute```.
2. В силу того, что в кампусе не нужна такая гранулярность, как она была нужна в ДЦ во время миграции между устройствами разных производителей с сохранением сервиса, мы упрощаем конфигурацию в следующем:
   1. Вместо VLAN-based используем VLAN-Aware bundle services;
   2. Вместо ESI per IFL используем ESI per IFD.

***Многие так же могут заметить, что используется dataplane MPLS для EVPN, а не VXLAN, причина кроется в следующем:***
1. У нас уже есть MPLS backbone и многие сервисы работают именно поверх MPLS, поэтому нам проще и быстрее предоставить сервис для пользователей в кампусе используя MPLS;
2. На площадке всего 2 устройства способных работать с EVPN/VXLAN и/или EVPN/MPLS, поэтому, чтобы не делать склеивание dataplane VXLAN/MPLS на этих самых устройствах, нам проще использовать только MPLS;
3. Исходя из предыдущего пункта, мы так же снижаем вероятность потенциального отказа вызванного ошибкой в ПО, когда используем меньшее количество протоколов.

## Топология

![Topology](/images/2020-10-03-juniper-ex9200-evpn-collapsed-core-mh-aa-topology.png)

## Плюсы и минусы в сравнении с MC-LAG

Я не могу пропустить это сравнение, так как оно все же может быть полезным.

**Плюсы**:
1. Отсутствие необходимости в дополнительном протоколе резервирования Ethernet - MC-LAG/Virtual-Chassis или других;
2. Отсутствие необходимости в дополнительном протоколе резервирования шлюза - VRRP или других;
3. Возможность использовать конфигурацию Active-Active как для L2, так и для L3 трафика;
4. Возможность горизонтального масштабирования (более двух устройств), ***но это не кейс для кампуса***, однако все равно плюс;
5. В случае необходимости нативными средствами EVPN (если другая сторона поддерживает EVPN/MPLS) возможно растянуть L2 домен;
6. Механизм ARP-Suppression который помогает сократить количество BUM трафика в сети передачи данных;
7. Сокращается количество запущенных протоколов на устройстве, в следствии чего уменьшается вероятность перебоя в работе устройства из-за возможной ошибки в одном из протоколов.

**Минусы**:
1. Невозможно использовать Active-Standby для L2 и для L3 трафика одновременно. Да, можно поменять Local-preference для L3 и сделать EVPN DF preference-based, но не все можно настроить руками и для этого нужен дополнительный внешний механизм - скрипты on-box/eem и прочее;
2. Новый протокол, еще недостаточно проверен временем и в реализации разных производителей до сих пор встречаются мажорные баги;
3. В дополнение к предыдущему пункту - не все инженеры могут быть готовы к поиску и исправлению проблем в сети передачи данных на базе EVPN; 
4. ***Невозможно обеспечить ZTP для нижестоящих устройств*** (серверов или коммутаторов доступа) в некоторых случаях ***без предварительного конфигурирования Attachment Circuit*** (AC) и перенастройки его в "боевой" режим после успешного провижининга нижестоящих устройств;
5. ***Невозможно использовать динамическую маршрутизацию на IRB с anycast-gateway***. Но, в качестве примера, ***это ограничение можно обойти, добавив индивидуальные IP адреса на каждый IRB как prefered***.

> Если что-то вдруг забыл указать, напишите в ЛС, добавлю.

## Пример конфигурации

Чтобы не смущать Вас, уважаемый читатель, большим куском конфигурации, сообщество подсказало разбить конфигурацию на части, с объяснениями таковых. В примере конфигурация AR01 будет разбита на части с объяснениями, а конфигурация AR02 будет приведена итоговая, одной не делимой чатсью.

> ***Конфигурация может изменятся по мере реализации новых возможностей производителем***.

> **Особенности:** Конфигурация тестировалась на Junos 19.3R3.6 и на 19.4R3.11, так как **начиная с Junos 19.3R1 добавили возможность использовать DHCP-Relay в EVPN/MPLS**.

> **Важно:** В версии ПО 19.3R3.6 была обнаружена проблема с "залипанием" MAC-адреса в kernel и как следствие невозможности перенаправления трафика к этому MAC. Из-за этого в данный момент используется версия ПО 19.4R3.11, на которой пока не обнаружено никаких проблем.

> **Примечание:** Часть важных опций прокомментирована и конфигурация, не имеющая отношение к делу, удалена. А также у меня есть привычка экономить и унифицировать конфигурацию через apply-groups, далее в конфигурации еще не однократно это будет видно.

**AR01**

***Версия ПО:***

```
version 19.4R3.11;
```

***Часть конфигурации, описывающая группу настройки OSPF интерфейсов:***
В данной части конфигурации нет прямого отношения к EVPN, но, поскольку используется MPLS "перекидка" в underlay между AR01 и AR02, а так же L3VPN для обеспечения доступа к сервисам через MPLS backbone, то, нам пригодится BFD для детектирования доступности связности по L3 в underlay между напрямую подключенными устройствами, чтобы быстрее изъять проблемный next-hop из FIB (если есть backup next-hop). Поскольку для BGP основное правило работы связано с доступностью next-hop, то можно позволить не использовать BFD в overlay, но мы к этому еще вернемся. Так же тут есть опция ```node-link-protection```, она нужна чтобы избегать проблемное устройство и ***по возможности*** инсталлировать в FIB backup next-hop другого устройства (про Remote-LFA и PQ-Node Вы можете ознакомится самостоятельно на сайте производителя). По возможности, потому что если не будет доступен ```node-link-protection```, то будет использован классический ```link-protection``` при помощи опции ```node-link-degradation```.

```
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
}
```

***Часть конфигурации, описывающая настройки RI VRF:***
В данной части конфигурации описана группа с базовыми настройками L3VPN инстанса, такими как ```instance-type```, ```multipath```, ```protect core``` (он же BGP PIC), ```auto-export``` и ```vrf-table-lable```. Для тех, кто не знаком с Junos, Вы можете ознакомится самостоятельно со всеми опциями на сайте производителя.

```
groups {
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
}
```

***Часть конфигурации, описывающая настройки DHCP-Relay:***
В данной части конфигурации описана группа для функционирования DHCP-Relay, мы ведь все же про кампус говорим.

```
groups {
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
}
```

***Часть конфигурации, описывающая настройки AC для EVPN A/A:***
В данной части конфигурации описана группа AC для EVPN в сценарии all-active для подключения коммутаторов доступа. Удобно описать одну группа и делать изменения именно в ней, чтобы применить настройки для всех интерфейсов, на которые применена данная группа, например добавить/удалить VLAN. Из важного ***стоит обратить внимание на указание наследование ESI от LACP system-id*** при помощи ```esi {
                    auto-derive {
                        lacp;
                    }}``` и на то что ***ESI указан в контексте IFD, а не IFL***, как я делал это в предыдущей заметке, потому что в кампусе нам не нужна такая гранулярность, как была нужна ранее.

```
groups {
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
}
```

***Часть конфигурации, описывающая настройки RI EVPN:***
В данной части конфигурации описана группа с базовыми настройками EVPN инстанса, такими как ```instance-type```, ```remote-ip-host-routes```, ```label-allocation per-instance```, ```default-gateway do-not-advertise``` и ```duplicate-mac-detection```.
Опция ```remote-ip-host-routes``` позволяет оптимизировать входящий трафик на шлюз, если это нам будет нужно.
Опция ```default-gateway do-not-advertise``` выключает анонсирование BGP default gateway extended community, а опция ```duplicate-mac-detection``` позволяет бороться с MAC-move.
***Хочу сразу заметить, что данная группа создана для конфигурации EVPN VLAN-based инстансов на всякий случай***, а у нас же будет использоваться один единственный инстанс EVPN VLAN-aware bundle о котором будет информация далее.

```
groups {
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
}
```

***Просто полезная часть конфигурации для кампусной сети в контексте ```system```, (Вы можете ознакомится самостоятельно со всеми опциями на сайте производителя) не имеющая прямого отношения к EVPN:***

```
system {
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
    ddos-protection {
        global {
            flow-detection;
        }
        protocols {
            dhcpv4 {
                aggregate {
                    flow-level-detection {
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
}
```

***Часть конфигурации, описывающая настройки MPLS интерфейсов:***

```
interfaces {
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
}
```

***Часть конфигурации, описывающая настройки физических интерфейсов*** для подключения коммутаторов доступа, а также логическое выделение количества LAG интерфейсов на шасси:
Стоит обратить внимание на ```hold-time up 120000``` - это сделано для того, чтобы предотвратить возможную потерю трафика, которая теоретически может быть вызвана следующими событиями:
1. Коммутатор доступа будет включен и появится "свет" в волокне на RX, а также LACP PDU;
2. Еще не будут до конца запущены и синхронизированы все протоколы между коммутаторами доступа (не забываем, коммутаторы доступа все же собраны в Virtual-Chassis).

А опция ```down 1``` помогла избежать не правильного программирования PFE при отключении "света" на RX в волокне в другом кейсе, связанным с MC-LAG, поэтому осталось как "наследие" навсегда с нами.

```
chassis {
    aggregated-devices {
        ethernet {
            device-count 24;
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
}
```

***Часть конфигурации, описывающая логические настройки интерфейсов*** для подключения коммутаторов доступа:
Стоит обратить внимание, что ***для каждого LAG интерфейса в контексте LACP указан уникальный system-id, так как ESI будет напрямую унаследован из него***. Не стоит забывать, что ***каждый коммутатор доступа, с точки зрения EVPN, является подключенным к изолированному ES и должен иметь уникальный ESI***.

```
interfaces {
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
}
```

***Часть конфигурации, описывающая настройки шлюза по умолчанию:***
Тут ничего не обычного в контексте EVPN all-active - вручную присвоенные ***одинаковые MAC адреса и IPv4 адреса шлюза по умолчанию на AR01 и AR02***, но уникальные для каждого bridge-domain.

```
interfaces {
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
}
```

***Часть конфигурации, описывающая адрес lo0.0 интерфейса и фильтрацию для защиты Control Plane*** (Вы можете ознакомится самостоятельно со всеми опциями на сайте производителя, так как описание фильтров выходит за рамки написания данной заметки):

```
interfaces {
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
```

***Часть конфигурации, описывающая балансировку per-flow с избеганием возможной поляризации трафика:***

```
forwarding-options {
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
}
routing-options {
    forwarding-table {
        export Load_balance;
    }
}
```

***Часть конфигурации, описывающая BGP политики на import/export, а так же используемые RT:***

```
policy-options {
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
```

***Часть конфигурации, описывающая RI EVPN VLAN-aware bundle:***
1. VLAN-aware bundle services можно сконфигурировать только с типом инстанса ```instance-type virtual-switch```;
2. Опция ```remote-ip-host-routes``` позволяет оптимизировать входящий трафик на шлюз, если это нам будет нужно;
3. Опция ```label-allocation per-instance``` позволяет сократить утилизацию LFIB;
4. Опция ```extended-vlan-list``` с указанием vlan-id необходима для корректной работы EVPN VLAN-aware bundle services;
5. Опция ```default-gateway do-not-advertise``` выключает анонсирование BGP default gateway extended community;
6. Опция ```duplicate-mac-detection``` позволяет бороться с MAC-move;
7. Перечень логических (LAG) интерфейсов для подключения коммутаторов доступа;
8. В ```switch-options``` мы задаем требуемые для нас лимиты, ***по умолчанию 5120***, чего конкретно нам в данном случае недостаточно;
9. ```vrf-target``` должен быть одинаковый между AR01 и AR02 в рамках конкретного инстанса;
10. Указание bridge-domain (vlans) и routing-interface (l3-interface) в Enterprise style, vlan-id.

```
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
}
```

***Часть конфигурации, описывающая RI VRF, с указанием L3 интерфейсов, ранее настроенных групп конфигураций и BGP политик:***

```
routing-instances {
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
```

***Часть конфигурации, описывающая параметры глобальные параметры маршрутизации:***
1. Опция ```route-distinguisher-id``` позволяет автоматически назначать route-distinguisher для каждого RI на основе заранее установленного значение, которое равняется RID;
2. ***Опция ```dynamic-list-next-hop``` нам нужна для EVPN, позволяет улучшить сходимость сети***;
3. ***Опция ```chained-composite-next-hop``` нам нужна так же для EVPN***, без которой функционирование EVPN не будет возможным. Дополнительно включена и для L3VPN;
4. RID;
5. ASn.

```
routing-options {
    route-distinguisher-id 10.100.0.96;
    forwarding-table {
        dynamic-list-next-hop;
        chained-composite-next-hop {
            ingress {
                evpn;
                l3vpn;
            }
        }
    }
    router-id 10.100.0.96;
    autonomous-system 65535;
}
```

***Часть конфигурации, описывающая общие параметры для обеспечения работы динамической маршрутизации и MPLS*** (описание всех пунктов выходит за рамки написания данной заметки):

```
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
}
```

***Часть конфигурации, необходимая для правильной работы EVPN Collapsed Core:***
Так как у нас BGP сессия с AF EVPN построена между двумя напрямую соединенными устройствами (на RRs нет смысла отдавать все маршруты EVPN) и если вдруг одно из устройств будет недоступно по тем или иным причинам, порвется BGP сессия, как следствие на "живом" устройстве отключаться все ES в сторону коммутаторов доступа, а это нам не нужно.

```
protocols {
    evpn {
        no-core-isolation;
    }
}
```

***Часть конфигурации, описывающая настройки BGP:***
1. ```vpn-apply-export``` переопределяет выбор маршрутов BGP полученных из RI VRF, не обязателен, но настроен заранее, так как на практике из AR могут попросить сделать BGP Peering Point :);
2. Группа ```internal``` - iBGP, нужна для обеспечения функционирования L3VPN, в этой группе так же есть AF rtarget для снижения нагрузки на Control Plane, все соседи являются RR;
3. Группа ```evpn-collapsed-core``` - iBGP, нужна для обеспечения функционирования EVPN Collapsed Core, единственным соседом внутри которой указано одноранговое устройство на площадке. Дополнительно включен BFD для быстрого детектирования проблем с соседом, целью которого является ускоренние отзыва префиксов EVPN;
4. Параметр ```local-preference``` на обоих AR одинаковый для обеспечения работы из других сегментов multipath к обоим AR;
5. ```graceful-restart``` выключен для ускорения сходимости сети в случае аварии, так как в L3VPN для сессий с RRs не настроен BFD для экономии ресурсов Control Plane виртуальных RRs;
6. Собственно ```multipath``` для обеспечения работы multipath;
7. Опция ```multipath-build-priority``` позволяет выбрать экономить ли ресурсы Control Plane во время расчета multipath или нет.

```
protocols {
    bgp {
        vpn-apply-export;
        group internal {
            type internal;
            local-address 10.100.0.96;
            family inet-vpn {
                unicast;
            }
            family route-target;
            peer-as 65535;
            local-as 65535;
            neighbor 10.100.0.243;
            neighbor 10.100.0.244;
        }
        group evpn-collapsed-core {
            type internal;
            local-address 10.100.0.96;
            family evpn {
                signaling;
            }
            family route-target;
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
        graceful-restart {
            disable;
        }
        multipath;
        multipath-build-priority {
            low;
        }
    }
}
```

**AR02**

***Конфигурация целиком:***

```
version 19.4R3.11;
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
                    flow-level-detection {
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
                unicast;
            }
            family route-target;
            peer-as 65535;
            local-as 65535;
            neighbor 10.100.0.243;
            neighbor 10.100.0.244;
        }
        group evpn-collapsed-core {
            type internal;
            local-address 10.100.0.96;
            family evpn {
                signaling;
            }
            family route-target;
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
<b id="f2">2</b>. [Evolved Campus Core:
An EVPN Framework for Campus Networks](https://www.juniper.net/assets/us/en/local/pdf/nxtwork/network-infrastructure-evolved-campus-core.pdf) [↩](#a2)<br/>
<b id="f3">3</b>. [Understanding When to Disable EVPN-VXLAN Core Isolation](https://www.juniper.net/documentation/en_US/junos/topics/concept/evpn-vxlan-core-isolation-disabling.html) [↩](#a3)<br/>
