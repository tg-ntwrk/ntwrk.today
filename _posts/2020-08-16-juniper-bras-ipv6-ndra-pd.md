---
layout: post
title: "Конфигурирование Juniper BRAS для IPoE клиентов с оспользованием Dual Stack и NDRA-PD"
tags: juniper bras ipv6 dual-stack ndra-pd
author: "github:kmisak"
---


Конфигурирование Juniper BRAS для IPoE клиентов с оспользованием Dual Stack и NDRA-PD

## Вместо предисловия

IPv6 хоть и не так быстро как хотелось, но расширяет свое присутствие в Интернет, и рано или поздно сетевым инженерам интернет провайдеров придется озаботиться его конфигурированием на своем оборудовании. Являясь администратором довольно большой клиентской базы, подключенной по GPON, я довольно рано озаботился его внедрением и теперь хочу поделиться своим опытом и конечно конфигурацией.

## Немного о технологии доступа и принятых решениях:

- /48 на клиента. Здесь учитывались рекомендации RIPE-690. Жадничать не стоит, это первое дело при внедрении IPv6, всегда надо смотреть в первую очередь на удобство администрирования, а не на экономию адресного пространства. Все привычки экономить, приобретенные за годы жизни с IPv4 надо пересматривать.

- /32 на BRAS. Математика тут такая - у нас Juniper MX480, который официально поддерживает максимум 128К сабскрайберов. Разделим это число на два, так как не стоит сильно верить вендору, даже джуну. Итого 64К /48 это /32. На всякий случай оставляем еще один /32 рядом с выделенным блоком свободным - мало ли, вдруг придется все-таки расти на одном BRAS до 128К. RIPE по умолчанию выделяет /32 для LIR, но вообще без разговоров дает /29 стоит только попросить. Я так и сделал. Это 16 /32 - должно хватить намного и надолго.

- Версия Junos сейчас в проде - 17.3R3-S3. Версии выше тоже подойдут, но я при сильной любви ставить последнии версии джуноса на РЕ маршрутизаторах достаточно консервативен в случаях с BRAS. Настоятельно рекомендуется версия, которая поддерживает dual-stack-group, тоесть начиная с 17.3R1, так как в противном случае IPv4 и IPv6 сессии каждого клиента будут считаться как две разные, что сразу приводит к излишним сложностям и на стороне биллинга, и с точки зрения использования ресурсов, а главное - сразу удваивается нужное количество лицензий, которые стоят денег.

- Самое главное решение и причина написания статьи - в начале была выбрана схема доступа DHCPv6 IA_NA and DHCPv6 Prefix Delegation, тоесть и адрес WAN интерфейса клиентского оборудования и выделяемый клиенту префикс раздавался DHCP сервером.

- Схема как это все работает (картинку я позаимствовал с сайта Juniper, надеюсь простят):

  ![IA_NA_PD](/images/IA_NA_PD.gif)


Давайте посмотрим конфигурацию, я прокомментировал все важные вещи. Конфигурация из прода, изменены только имена, адреса, пароли заменены на хххх.

```
groups {
    arp-policer-64k {
        #Juniper по умолчанию на каждом интерфейсе полисит arp трафик, при большом
        #количестве клиентов надо полисер увеличивать, а то будут проблемы
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
        dhcp-local-server {
            dhcpv6 {
                group DS_v6 {
                    overrides {
                        #пул делегированных адресов
                        delegated-pool v6-prefix-delegation;
                        #общая часть конфига IPv4 и IPv6 в дуал стек группе
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
                    #общая часть конфига IPv4 и IPv6 в дуал стек группе
                    dual-stack DS;
                }
                interface ae1.106;
            }
            dual-stack-group DS {
                authentication {
                    password xxxxx;
                    username-include {
                        #мы используем в качестве имени пользователя mac адрес
                        mac-address;
                    }
                }
                dynamic-profile user-profile-ds;
                classification-key {
                    mac-address;
                }
                #если клиент не авторизовался для IPv4 IPv6 сессия тоже считается неавторизованной
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
                    proxy-arp;
                    demux-options {
                        underlying-interface "$junos-underlying-interface";
                    }
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
                    #Использовать DHCPv6 для получения конфигурации адреса
                    managed-configuration;
                    other-stateful-configuration;
                }
            }
        }
    }
}
access-profile access;
interfaces {
    ae1 {
        description ->AGG;
        flexible-vlan-tagging;
        mtu 9192;
        encapsulation flexible-ethernet-services;
        unit 106 {
            #см. описание группы
            apply-groups arp-policer-64k;
            description GPON;
            demux-source [ inet inet6 ];
            vlan-id 106;
            family inet {
                #боремся против спуфинга адресов
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
    policer arp-64k {
        filter-specific;
        if-exceeding {
            bandwidth-limit 64k;
            burst-size-limit 8k;
        }
        then discard;
    }
}
access {
    profile access {
        #при проблемах с RADIUS сервером пускать клиентов без авторизации, зачем нам злые клиенты пока мы сервер чиним?
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
                secret "xxxxxxxxxxxxxxxxxxxx"; ## SECRET-DATA
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
            #когда закончатся адреса в этом пуле, использовать пул GPON2
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
    domain {
        map default {
            aaa-routing-instance default;
            access-profile access;
            strip-domain;
        }
    }
}
```

Что в последнем решении плохого? В самом деле ничего, но есть некоторые ньюансы:

- Глобально доступные адреса на WAN интерфейсе клиентов. Не секрет, что в подавляющей своей массе производители CPE не сильно думают о безопасности, оставляя бэкдоры, слабые пароли и лишние сервисы. Достаточно упомянуть, что подавляющее большинство участников ботнетов, которые DDoS тут и там всех кого ни попадя - это домашние рутеры и CPE. Конечно можно возразить, что надо вешать фильтр и разрешать доступ к этому пулу адресов только кому нужно, но это по сравнению с найденным решением требует лишних ресурсов и усложняет конфигурацию. В общем, не хочется выставлять голый зад в Интернет.

- Реализация DHCP клиента у подавляющего большинства CPE тоже оставляет лучшего, так что хотелось бы его избежать где только возможно.

После полугода работы этой схемы и сбора подробной информации о поведении и наших CPE, и джуниперов я решил заменить схему доступа на NDRA PD, тоесть адрес WAN CPE получает сразу при neighbor discovery, потом по DHCP запрашивает префикс для клиента. Также решено было раздавать на WAN адреса из диапазона адресов ULA - fc00::/7. Это аналог широко известных из мира IPv4 приватных адресов, они не маршрутизируемы в Интернет, только в пределах организации. Этим решаем проблему видимости CPE для всего интернета, не теряя самому доступа к оборудованию при необходимости. 

Вот как это описывает в своей документации джунипер:

  ![NDRA_PD](/images/NDRA_PD.gif)

В результате изменений пришли к такой конфигурации:

```
system {
    services {
        dhcp-local-server {
            dhcpv6 {
                group DS_SLAAC_PD {
                    overrides {
                        delegated-pool v6-prefix-delegation;
                        dual-stack DS_NDRA_PD;
                    }
                    interface ae1.2005;
                }
            }
            group DHCP-NDRA {
                overrides {
                    dual-stack DS_NDRA_PD;
                }
                interface ae1.2005;
            }
            dual-stack-group DS_NDRA_PD {
                authentication {
                    password xxxxxx;
                    username-include {
                        mac-address;
                    }
                }
                dynamic-profile user-profile-nrda-pd;
                on-demand-address-allocation;
                classification-key {
                    mac-address;
                }
                protocol-master inet;
            }
        }
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
                    proxy-arp;
                    demux-options {
                        underlying-interface "$junos-underlying-interface";
                    }
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
interfaces {
    ae1 {
        unit 2005 {
            description GPON;
            demux-source [ inet inet6 ];
            vlan-id 2005;
            family inet {
                rpf-check   fail-filter RPF-ALLOW-DHCP;
                unnumbered-address lo0.0 preferred-source-address 192.0.2.1;
            }
            family inet6 {
                #это один из ньюансов, не надо указывать никакого адреса, не знаю почему
            }
        }
    }
access {
    address-assignment {
        neighbor-discovery-router-advertisement ndra-prefixes;
        pool ndra-prefixes {
            family inet6 {
                prefix fc00:0001::/48;
                range R1 prefix-length 64;
                dhcp-attributes {
                    domain-name rtarmenia.am;
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

Вообще по правилам использования ULA адресов, описанных в RFC4193, надо генерировать уникальный префикс, например с помощью такого онлайн инструмента, ссылка на который есть ниже. Но лично для меня важнее было удобство запоминания и кодирования структуры сети в адресе, так что я этим требованием пренебрег. Если есть какая-то причина, по которой вы считаете, что это неправильно - приходите в чат, подискутируем.

## Ссылки
<b id="f1">1</b>. [RIPE-690](https://www.ripe.net/publications/docs/ripe-690) [↩](#a1)<br/>
<b id="f2">2</b>. [IPv6 addressing subscriber access designs](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/ipv6-addressing-subscriber-access-designs.html) [↩](#a2)<br/>
<b id="f3">3</b>. [RFC4193](https://tools.ietf.org/html/rfc4193) [↩](#a3)<br/>
<b id="f4">4</b>. [ULA Range Generator](https://www.ultratools.com/tools/rangeGenerator) [↩](#a4)<br/>