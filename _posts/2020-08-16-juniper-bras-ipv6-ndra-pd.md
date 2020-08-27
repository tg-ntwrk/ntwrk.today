---
layout: post
title: "Конфигурирование Juniper BRAS для IPoE клиентов с оспользованием Dual Stack и NDRA-PD"
tags: juniper bras ipv6 dual-stack ndra-pd ipoe
author: "kmisak"
---

Конфигурирование Juniper BRAS для IPoE клиентов с оспользованием Dual Stack и NDRA-PD

## Вместо предисловия

IPv6 продолжает расширять свое присутствие, рано или поздно сетевым инженерам интернет-провайдеров придется озаботиться его конфигурированием на оборудовании. Статья описывает опыт внедрения на большой абонентской базе, подключенной по [GPON](https://en.wikipedia.org/wiki/Passive_optical_network) <sup id="a1">[1](#f1)</sup>.

### Технологии и принятые решения:

Выделение /48 перифкса на клиента. Учитывались рекомендации [RIPE-690](https://www.ripe.net/publications/docs/ripe-690) <sup id="a2">[2](#f2)</sup>, при исопльзовании IPv6 первично должно быть удобство администрирования, а не экономия адресного пространства.

Выделение /32 на один [BRAS](https://en.wikipedia.org/wiki/Broadband_remote_access_server) <sup id="a3">[3](#f3)</sup>, для начала. Используется Juniper MX480, официально поддерживающий до 256.000 [Dual Stack](https://en.wikipedia.org/wiki/IPv6#Dual-stack_IP_implementation) <sup id="a4">[4](#f4)</sup> пользователей. Адреса выделяются порциями по /32, это 65.536 пользователей. RIPE NCC по умолчанию выделяет /32 для LIR, но можно попросить без особых обоснований и /29, это 16 x /32.

Стратегия выделения адресов:
* Блоки адресов для инфраструктуры резервируются с начала выделенного адресного пространства;
* Блоки адресов для клиентов резервируются с конца выделенного адресного пространства;
* Если клиенту выделяется блок какого-либо размера, рядом резервируется дополнительный блок такого же размера, если клиенту в будущем понадобятся дополнительные адреса. Это удобно и с точки зрения агрегации, два меньших блока легко можно агрегировать в один, при этом адресация клиента не будет перемещаться по IP plan.

  ![Addressing_sceme](/images/ipv6-addressing-scheme.png)

На момент написания статьи используемая на [BRAS](https://en.wikipedia.org/wiki/Broadband_remote_access_server) <sup id="a3">[3](#f3)</sup> версия [Junos® OS 17.3R3-S3](https://kb.juniper.net/InfoCenter/index?page=content&id=TSB17512&cat=&actp=LIST) <sup id="a5">[5](#f5)</sup>.

> **Внимание**: Настоятельно рекомендуется версия с поддержкой [dual-stack-group](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/system-services-dhcp-local-server-dual-stack-group.html) <sup id="a6">[6](#f6)</sup> (начиная с [Junos® OS 17.3R1](https://www.juniper.net/documentation/en_US/junos/information-products/topic-collections/release-notes/17.3/junos-release-notes-17.3r1.pdf) <sup id="a7">[7](#f7)</sup>), иначе IPv4 и IPv6 сессии каждого клиента будут идентифицированы отдельно. Это приведёт к излишним сложностям как на стороне [BSS](https://en.wikipedia.org/wiki/Business_support_system) <sup id="a8">[8](#f8)</sup>, так и с точки зрения использования ресурсов, дополнительное удваивается требуемое количество лицензий.

## DHCPv6 IA_NA, DHCPv6 Prefix Delegation

Изначально была выбрана схема доступа [DHCPv6 IA_NA and DHCPv6 Prefix Delegation](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dhcpv6-iana-prefix-delegation-addressing.html) <sup id="a9">[9](#f9)</sup>, когда адрес [WAN](https://en.wikipedia.org/wiki/Wide_area_network) <sup id="a10">[10](#f10)</sup> интерфейса [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> и выделяемый клиенту префикс раздаются DHCP сервером:

  ![IA_NA_PD](/images/IA_NA_PD.gif)

### Junos® OS конфигурация

Конфигурация [BRAS](https://en.wikipedia.org/wiki/Broadband_remote_access_server) <sup id="a3">[3](#f3)</sup> с использованием [DHCPv6 IA_NA and DHCPv6 Prefix Delegation](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dhcpv6-iana-prefix-delegation-addressing.html) <sup id="a9">[9](#f9)</sup>. Изменены имена, адреса, пароли замаскированы.

Блок общих настроек

```ruby
groups {
    arp-policer-64k {
        # По умолчанию на каждом интерфейсе полисится ARP трафик, при большом
        # количестве клиентов количество записей необходимо увеличить
        interfaces {
            <*> {
                unit <*> {
                    family inet {
                        policer {
                            arp arp-64k;
                        }
                    }
                }
            }
        }
    }
}
system {
    configuration-database {
        max-db-size 314572800;
    }
    dynamic-profile-options {
        versioning;
    }
}
```

Определена группа для использования ARP полисера на интерфейсе в сторону клиентов, так как полисер по умолчанию слишком строг для даже небольшого количества клиентов. Подробная статья про [особенности работы ARP в Junos® OS](https://ntwrk.today/2020/01/16/juniper-arp.html) <sup id="a12">[12](#f12)</sup>.

В этом же блоке задается размер базы конфигурации и включается версионирование динамических профилей, большое подробностей об этом можно найти в документации [Versioning for Dynamic Profiles](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dynamic-profiles-versioning.html) <sup id="a13">[13](#f13)</sup>.

Конфигурация сервисов DHCPv4, DHCPv6 и [Dual Stack группы](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/system-services-dhcp-local-server-dual-stack-group.html) <sup id="a6">[6](#f6)</sup>, конфигурация процесса управления клиентами. [Dual Stack группы](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/system-services-dhcp-local-server-dual-stack-group.html) <sup id="a6">[6](#f6)</sup> удобны для выноса общих настроек в отдельный блок, создавая более читаемую конфигурацию:

```ruby
system {
    dhcp-local-server {
        dhcpv6 {
            group DS_v6 {
                overrides {
                    # Pool делегированных адресов
                    delegated-pool v6-prefix-delegation;
                    # Общая часть конфигурации IPv4 и IPv6 в Dual Stack группе
                    dual-stack DS;
                }
                interface ae1.106;
            }
        }
        pool-match-order {
            ip-address-first;
        }
        group DHCPv4 {
            overrides {
                # Общая часть конфигурации IPv4 и IPv6 в Dual Stack группе
                dual-stack DS;
            }
            interface ae1.106;
        }
        dual-stack-group DS {
            authentication {
                password xxxxx;
                username-include {
                    # В качестве имени пользователя используется MAC адрес
                    mac-address;
                }
            }
            dynamic-profile user-profile-ds;
            classification-key {
                mac-address;
            }
            # Если клиент не авторизовался для IPv4,
            # IPv6 сессия тоже считается неавторизованной
            protocol-master inet;
        }
    }
    subscriber-management {
        gres-route-flush-delay;
        enforce-strict-scale-limit-license;
        enable;
    }
}
processes {
    routing failover other-routing-engine;
    dhcp-service {
        failover other-routing-engine;
    }
}
```

Динамический профиль:

```ruby
dynamic-profiles {
    user-profile-ds {
        routing-instances {
            "$junos-routing-instance" {
                interface "$junos-interface-name";
            }
        }
        interfaces {
            demux0 {
                unit "$junos-interface-unit" {
                    no-traps;
                    # Proxy ARP для возможности клиентским CPE обнаруживать друг друга,
                    # OLT по умолчанию изолируют клиентов друг от друга даже в пределах одного VLAN
                    proxy-arp;
                    demux-options {
                        underlying-interface "$junos-underlying-interface";
                    }
                    # Если используются агрегированные интерфейсы, это помогает относить каждого клиента
                    # к одному их интерфейсов агрегата, иначе полисеры буду работать неправильно
                    targeted-distribution;
                    family inet {
                        rpf-check fail-filter RPF-ALLOW-DHCP;
                        demux-source {
                            $junos-subscriber-ip-address;
                        }
                        unnumbered-address "$junos-loopback-interface";
                    }
                    family inet6 {
                        rpf-check fail-filter RPF-ALLOW-DHCPv6;
                        demux-source {
                            "$junos-subscriber-ipv6-address";
                        }
                        unnumbered-address "$junos-loopback-interface";
                    }
                }
            }
        }
        protocols {
            router-advertisement {
                interface "$junos-interface-name" {
                    # Использование DHCPv6 для получения конфигурации адреса
                    managed-configuration;
                    other-stateful-configuration;
                }
            }
        }
    }
}
```

Конфигурация интерфейсов:
```ruby
interfaces {
    ae1 {
        description ->AGG;
        flexible-vlan-tagging;
        mtu 9192;
        encapsulation flexible-ethernet-services;
        unit 106 {
            # См. описание группы
            apply-groups arp-policer-64k;
            description GPON;
            demux-source [ inet inet6 ];
            vlan-id 106;
            family inet {
                # Борьба со спуфингом адресов
                rpf-check fail-filter RPF-ALLOW-DHCP;
                unnumbered-address lo0.0 preferred-source-address 192.0.2.1;
            }
            family inet6 {
                unnumbered-address lo0.0 preferred-source-address 2001:2:1:a::1;
            }
        }
    }
    lo0 {
        unit 0 {
            apply-groups protect-re-group;
            family inet {
                address 192.0.2.1/32;
            }
            family inet6 {
                address 2001:2:1:a::1/128;
            }
        }
    }
}
```
Можно отметить включение [RPF Check](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/interfaces-configuring-unicast-rpf.html) <sup id="a14">[14](#f14)</sup> на интерфейсах в сторону клиентов. Механизм [RPF Check](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/interfaces-configuring-unicast-rpf.html) <sup id="a14">[14](#f14)</sup> [рекомендован MANRS](https://www.manrs.org/isps/guide/antispoofing/) <sup id="a15">[15](#f15)</sup> как вклад каждого провайдера в борьбу со спуфингом адресов, ботнетами и паразитным трафиком.

Конфигурация фильтров:
```ruby
firewall {
    family inet6 {
        filter RPF-ALLOW-DHCPv6 {
            term ALLOW-DHCP-INET6 {
                from {
                    next-header udp;
                    source-port [ 546 547 ];
                    destination-port [ 546 547 ];
                }
                then accept;
            }
            term ALLOW-DHCP-ICMP6 {
                from {
                    next-header icmp6;
                    icmp-type [ router-solicit neighbor-solicit neighbor-advertisement ];
                }
                then accept;
            }
            term DISCARD-ALL {
                then discard;
            }
        }
        filter RPF-ALLOW-DHCP {
            term ALLOW-DHCP-BOOTP {
                from {
                    destination-port dhcp;
                }
                then accept;
            }
            term DISCARD-ALL {
                then {
                    discard;
                }
            }
        }
    }
    policer arp-64k {
        # При каждом назначении на интерфейс создаётся новый экземпляр полисера,
        # иначе будет использоваться один полисер на все интерфейсы,
        # соответственно и резаться будут все интерфейсы суммарно в 64 Кбит/с
        filter-specific;
        # Выделяем ARP 64 Кбит/с, разрешаем всплески до 8 КБайт
        if-exceeding {
            bandwidth-limit 64k;
            burst-size-limit 8k;
        }
        then discard;
    }
}
```

Фильтр позволяет пакетам DHCPv4 и DHCPv6 в качестве исключения обходить этот механизм, в противном случае клиенты не получат адресов. Также описан ARP полисер.

Конфигурация профилей доступа и пулов адресов:

```ruby
access-profile access;
access {
    profile access {
        # При неработосопособности RADIUS сервера разрешить клиентам работать без авторизации
        authentication-order [ radius none ];
        radius {
            authentication-server 198.51.100.2;
            accounting-server 198.51.100.2;
            options {
                coa-dynamic-variable-validation;
            }
        }
        radius-server {
            198.51.100.2 {
                port 1812;
                accounting-port 1813;
                secret "xxxxxxxxxxxxxxxxxxxx";
                retry 3;
            }
        }
        accounting {
            order radius;
            accounting-stop-on-failure;
            accounting-stop-on-access-deny;
            immediate-update;
            coa-immediate-update;
            update-interval 10;
            statistics time;
            send-acct-status-on-config-change;
        }
        service {
            accounting-order activation-protocol;
        }
    }
    profile auth_none {
        authentication-order none;
    }
    address-assignment {
        pool GPON {
            # Когда закончатся адреса в пуле GPON,
            # начать использовать пул GPON2
            link GPON2;
            family inet {
                network 10.2.0.0/19;
                range R1 {
                    low 10.2.0.10;
                    high 10.2.31.254;
                }
                dhcp-attributes {
                    maximum-lease-time 300;
                    grace-period 250;
                    domain-name xxxx;
                    name-server {
                        198.51.100.30;
                        198.51.100.3;
                    }
                    router {
                        10.2.0.1;
                    }
                }
            }
        }
        pool v6-prefix-wan {
            family inet6 {
                prefix 2001:2:1:a::/64;
                range R1 {
                    low 2001:2:1:a::2/128;
                    high 2001:2:1:a::ffff:ffff/128;
                }
                dhcp-attributes {
                    domain-name xxxx;
                    dns-server {
                        2001:2:0:53::30;
                        2001:2:0:53::31;
                    }
                }
            }
        }
        pool v6-prefix-delegation {
            family inet6 {
                prefix 2001:db8::/32;
                range prefix-range prefix-length 48;
            }
        }
    }
    domain {
        map default {
            aaa-routing-instance default;
            access-profile access;
            strip-domain;
        }
    }
}
```

Описаны два пула IPv6 адресов:
* Пул для [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> клиента на [WAN](https://en.wikipedia.org/wiki/Wide_area_network) <sup id="a10">[10](#f10)</sup> адрес;
* Пул адресов для клиентов блоками по /48.

Реализация такой конфигурации имеет некоторые ньюансы. Глобально доступные адреса на [WAN](https://en.wikipedia.org/wiki/Wide_area_network) <sup id="a10">[10](#f10)</sup> интерфейсе [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> клиентов: производители [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> мало заботятся о безопасности, оставляя бэкдоры, слабые пароли и лишние сервисы с уязвимостями. Подавляющее большинство участников ботнетов, используемых для разнообразных [DDoS](https://en.wikipedia.org/wiki/Denial-of-service_attack#Distributed_DoS) <sup id="a16">[16](#f16)</sup> атак - домашние маршрутизаторы и [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup>. Также имплементация DHCP клиента у большинства [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> недостаточно полноценна.

## NDRA-PD

В дальнейшем схема доступа была заменена на [NDRA-PD](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/ipv6-addressing-subscriber-access-designs.html#id-design-2-ipv6-addressing-with-ndra-and-dhcpv6-prefix-delegation) <sup id="a17">[17](#f17)</sup>, в которой [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> получает [WAN](https://en.wikipedia.org/wiki/Wide_area_network) <sup id="a10">[10](#f10)</sup> адрес сразу при [Neighbor Discovery](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) <sup id="a18">[18](#f18)</sup>, а затем по DHCP запрашивается префикс для клиента. Также было принято решение раздавать клиентским [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> на [WAN](https://en.wikipedia.org/wiki/Wide_area_network) <sup id="a10">[10](#f10)</sup> интерфейсы адреса из диапазона [ULA fc00::/7](https://en.wikipedia.org/wiki/Unique_local_address) <sup id="a19">[19](#f19)</sup>. Такая адресация маршрутизируется только в пределах организации и этим решается вопрос отсутствия видимости [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> в глобальной таблице маршрутизации без потери возможности управления [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment) <sup id="a11">[11](#f11)</sup> в рамках интернет-провайдера:

  ![NDRA_PD](/images/NDRA_PD.gif)

> **Внимание**: По правилам использования ULA адресов из [RFC4193](https://tools.ietf.org/html/rfc4193) <sup id="a20">[20](#f20)</sup> следует генерировать уникальный префикс. Для автора важнее удобство запоминания и кодирования структуры сети в адресе, рекомендацией из [RFC4193](https://tools.ietf.org/html/rfc4193) <sup id="a20">[20](#f20)</sup> было решено пренебречь.

### Junos® OS конфигурация

Конфигурация сервисов DHCPv4/DHCPv6 и [Dual Stack групп](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/system-services-dhcp-local-server-dual-stack-group.html) <sup id="a6">[6](#f6)</sup> идентична предыдущей, но указан другой динамический профиль:

```ruby
system {
    services {
        dhcp-local-server {
            dhcpv6 {
                group DS_SLAAC_PD {
                    overrides {
                        delegated-pool v6-prefix-delegation;
                        # Общая часть конфигурации IPv4 и IPv6 в Dual Stack группе
                        dual-stack DS_NDRA_PD;
                    }
                    interface ae1.2005;
                }
            }
            group DHCP-NDRA {
                overrides {
                    # Общая часть конфигурации IPv4 и IPv6 в Dual Stack группе
                    dual-stack DS_NDRA_PD;
                }
                interface ae1.2005;
            }
            dual-stack-group DS_NDRA_PD {
                authentication {
                    password xxxxxx;
                    username-include {
                        # В качестве имени пользователя используется MAC адрес
                        mac-address;
                    }
                }
                dynamic-profile user-profile-nrda-pd;
                on-demand-address-allocation;
                classification-key {
                    mac-address;
                }
                # Если клиент не авторизовался для IPv4,
                # IPv6 сессия тоже будет считается неавторизованной
                protocol-master inet;
            }
        }
    }
}
```

Динамический профиль:
```ruby
dynamic-profiles {
    user-profile-nrda-pd {
        routing-instances {
            "$junos-routing-instance" {
                interface "$junos-interface-name";
            }
        }
        interfaces {
            demux0 {
                unit "$junos-interface-unit" {
                    no-traps;
                    # Proxy ARP для возможности клиентским CPE обнаруживать друг друга,
                    # OLT по умолчанию изолируют клиентов друг от друга даже в пределах одного VLAN
                    proxy-arp;
                    demux-options {
                        underlying-interface "$junos-underlying-interface";
                    }
                    # Если используются агрегированные интерфейсы, это помогает относить каждого клиента
                    # к одному из интерфейсов агрегата, иначе полисеры буду работать неправильно
                    targeted-distribution;
                    family inet {
                        rpf-check fail-filter RPF-ALLOW-DHCP;
                        demux-source {
                            $junos-subscriber-ip-address;
                        }
                        unnumbered-address "$junos-loopback-interface";
                    }
                    family inet6 {
                        rpf-check fail-filter RPF-ALLOW-DHCPv6;
                        address $junos-ipv6-address;
                        demux-source {
                            "$junos-subscriber-ipv6-address";
                        }
                    }
                }
            }
        }
        protocols {
            router-advertisement {
                interface "$junos-interface-name" {
                    max-advertisement-interval 60;
                    min-advertisement-interval 10;
                    other-stateful-configuration;
                    default-lifetime 3600;
                    prefix $junos-ipv6-ndra-prefix;
                }
            }
        }
    }
}
```
IPv4 часть конфигурации не претерпела никаких изменений, для IPv6 удалена конфигурация с unnumbered-address, так как [BRAS](https://en.wikipedia.org/wiki/Broadband_remote_access_server) <sup id="a3">[3](#f3)</sup> на каждый интерфейс клиента должен выделять адрес из специального пула адресов NDRA. Поменялась конфигурация протокола router-advertisement, для раздачи адресов будет использован механизм [SLAAC](https://en.wikipedia.org/wiki/IPv6#Stateless_address_autoconfiguration_(SLAAC)) <sup id="a21">[21](#f21)</sup> и только делегированные префиксы будут выданы клиентам по DHCP.

Конфигурация интерфейсов:

```ruby
interfaces {
    ae1 {
        unit 2005 {
            description GPON;
            demux-source [ inet inet6 ];
            vlan-id 2005;
            family inet {
                # Борьба со спуфингом адресов
                rpf-check fail-filter RPF-ALLOW-DHCP;
                unnumbered-address lo0.0 preferred-source-address 192.0.2.1;
            }
            family inet6 {
                # Не требуется указывать адрес
            }
        }
    }
}
```

И новый пул:
```ruby
access {
    address-assignment {
        neighbor-discovery-router-advertisement ndra-prefixes;
        pool ndra-prefixes {
            family inet6 {
                prefix fc00:0001::/48;
                range R1 prefix-length 64;
                dhcp-attributes {
                    domain-name domain.tld;
                    dns-server {
                        2001:2:0:53::30;
                        2001:2:0:53::31;
                    }
                }
            }
        }
    }
}
```

## Ссылки
<b id="f1">1</b>. [Gigabit Passive Optical Network](https://en.wikipedia.org/wiki/Passive_optical_network) [↩](#a1)<br/>
<b id="f2">2</b>. [Best Current Operational Practice for Operators: IPv6 prefix assignment for end-users](https://www.ripe.net/publications/docs/ripe-690) [↩](#a2)<br/>
<b id="f3">3</b>. [Broadband Remote Access Server](https://en.wikipedia.org/wiki/Broadband_remote_access_server) [↩](#a3)<br/>
<b id="f4">4</b>. [Dual Stack](https://en.wikipedia.org/wiki/IPv6#Dual-stack_IP_implementation) [↩](#a4)<br/>
<b id="f5">5</b>. [Software Release Notification for Junos Software Service Release version 17.3R3-S3](https://kb.juniper.net/InfoCenter/index?page=content&id=TSB17512&cat=&actp=LIST) [↩](#a5)<br/>
<b id="f6">6</b>. [Configuring dual-stack-group (DHCP Local Server)
](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/system-services-dhcp-local-server-dual-stack-group.html) [↩](#a6)<br/>
<b id="f7">7</b>. [Junos® OS 17.3R1 Release Notes](https://www.juniper.net/documentation/en_US/junos/information-products/topic-collections/release-notes/17.3/junos-release-notes-17.3r1.pdf) [↩](#a7)<br/>
<b id="f8">8</b>. [Business Support System](https://en.wikipedia.org/wiki/Business_support_system) [↩](#a8)<br/>
<b id="f9">9</b>. [DHCPv6 IA_NA and DHCPv6 Prefix Delegation](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dhcpv6-iana-prefix-delegation-addressing.html) [↩](#a9)<br/>
<b id="f10">10</b>. [Wide Area Network](https://en.wikipedia.org/wiki/Wide_area_network) [↩](#a10)<br/>
<b id="f11">11</b>. [Customer-premises Equipment](https://en.wikipedia.org/wiki/Customer-premises_equipment) [↩](#a11)<br/>
<b id="f12">12</b>. [Особенности работы ARP протокола в Junos Network Operating System](https://ntwrk.today/2020/01/16/juniper-arp.html) [↩](#a12)<br/>
<b id="f13">13</b>. [Versioning for Dynamic Profiles](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dynamic-profiles-versioning.html) [↩](#a13)<br/>
<b id="f14">14</b>. [Configuring Unicast RPF check](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/interfaces-configuring-unicast-rpf.html) [↩](#a14)<br/>
<b id="f15">15</b>. [Anti-Spoofing – Preventing traffic with spoofed source IP addresses](https://www.manrs.org/isps/guide/antispoofing/) [↩](#a15)<br/>
<b id="f16">16</b>. [Distributed Denial-of-service Attack](https://en.wikipedia.org/wiki/Denial-of-service_attack#Distributed_DoS) [↩](#a16)<br/>
<b id="f17">17</b>. [NDRA-PD](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/ipv6-addressing-subscriber-access-designs.html#id-design-2-ipv6-addressing-with-ndra-and-dhcpv6-prefix-delegation) [↩](#a17)<br/>
<b id="f18">18</b>. [Neighbor Discovery Protocol](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) [↩](#a18)<br/>
<b id="f19">19</b>. [Unique Local Address](https://en.wikipedia.org/wiki/Unique_local_address) [↩](#a19)<br/>
<b id="f20">20</b>. [RFC4193: Unique Local IPv6 Unicast Addresses](https://tools.ietf.org/html/rfc4193) [↩](#a20)<br/>
<b id="f21">21</b>. [Stateless Address Autoconfiguration](https://en.wikipedia.org/wiki/IPv6#Stateless_address_autoconfiguration_(SLAAC)) [↩](#a21)<br/>
