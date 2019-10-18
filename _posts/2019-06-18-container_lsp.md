---
layout: post
title: "Динамическое управление пропускной способностью с использованием Container LSP (TE++)"
tags: mpls rsvp clsp
author: "somovis"
---

Динамическое управление пропускной способностью с использованием Container LSP \(TE++\).

## Почему TE++ и зачем это всё

Проблемы, из-за которых был придуман cLSP \(TE++\):

- **В сетях с отсутствием TE контроллера** существует [проблема bin-packing](https://www.ietf.org/proceedings/82/slides/pce-11.pdf), которая происходит из-за отсутствия на ingress маршрутизаторе информации об отдельных LSP и пропускной способности для каждого узла сети \(нет централизованного и полного видения состояния сети и всех её элементов, как на TE контроллере\). Конечно есть setup и hold приоритеты, но они работают ровно до того момента, пока не попадутся LSP с одинаковыми приоритетами и на транзитном маршрутизаторе не будет хватать пропускной способности
- Относительная сложность реализации ECMP
- Относительная сложность конфигурации

Решения проблем, которые предлагает cLSP \(TE++\):

- bin-packing - вместо того, чтобы сигнализировать один "толстый" LSP, маршрутизатор, с учетом текущей топологии и утилизации каналов, обсчитывает и сигнализирует несколько менее требовательных к полосе путей. В [draft](https://tools.ietf.org/html/draft-kompella-mpls-rsvp-ecmp-00) они называются sub-LSPs
- ECMP - sub-LSPs используются для балансировки. Каждый sub-LSP может использовать отличный путь к тому же egress маршрутизатору. Доступные для выбора методы балансировки по LSP \(единовременно возможен только один вариант\):
  - random
  - least-fill
  - most-fill

- Конфигурация - sub-LSPs не нужно настраивать отдельно \(как если бы мы настраивали обычный RSVP TE с несколькими LSP для решения проблемы bin-packing\). Настраивается "контейнер", в котором задаются параметры данного "контейнера", такие как:
  1. **Нормализация** - периодический процесс, когда предпринимается действие по настройке sub-LSPs, либо по настройке их пропускной способности, их количества или всего вместе. Процесс нормализации связан с процессом выборки \(sampling\) и периодически оценивает совокупное \(aggregate utilization\) использование контейнерного LSP
  2. **Номинальный LSP** - экземпляр контейнера LSP, который всегда присутствует
  3. **Дополнительный LSP** - экземпляры или sub-LSPs контейнера LSP, которые создаются или удаляются динамически. Автоматическая полоса пропускания \(auto-bw\) запускается для каждого из sub-LSPs, и каждый LSP изменяется в соответствии с трафиком, который он переносит, и параметрами конфигурации автоматической полосы пропускания. Совокупное \(aggregate utilization\) использование контейнерного LSP отслеживается путем суммирования пропускной способности для всех sub-LSPs
  4. **Minimum signaling-bandwidth** - минимальная полоса пропускания, с которой sub-LSPs сигнализируется во время нормализации или инициализации. Этот параметр может отличаться от минимальной полосы пропускания auto-bw
  5. **Maximum signaling-bandwidth** - максимальная полоса пропускания, с которой sub-LSPs сигнализируется во время нормализации или инициализации. Этот параметр может отличаться от минимальной полосы пропускания auto-bw
  6. **Merging-bandwidth** - задает нижний порог пропускной способности при использовании совокупной пропускной способности \(aggregate utilization\). Если совокупное использование падает ниже этого значения, ingress маршрутизатор принимает решение удалить один или несколько sub-LSP\(s\) во время нормализации
  7. **Splitting-bandwidth** - задает верхний порог пропускной способности при использовании совокупной пропускной способности \(aggregate utilization\). Если совокупное использование возрастает выше этого значения, ingress маршрутизатор принимает решение добавить один или несколько sub-LSP\(s\) во время нормализации
  8. В дополнение к Merging-bandwidth и Splitting-bandwidth маршрутизатор может принять решение ничего не менять, если не поменялась статистика за период для нормализации
  9. **Aggregate minimum-bandwidth** - сумма merging-bandwidth текущего активного sub-LSPs. Эта минимальная пропускная способность отличается от минимальной пропускной способности auto-bw
  10. **Aggregate maximum-bandwidth** - сумма splitting-bandwidth текущего активного sub-LSPs. Эта максимальная пропускная способность отличается от максимальной пропускной способности auto-bw

## Реализация Container LSP

Чтобы не быть переводчиком, рекомендую прочитать [документацию про реализацию](https://www.juniper.net/documentation/en_US/junos/topics/concept/dynamic-bandwidth-management-overview.html#jd0e505) Container LSP, а русскоязычным пользователям для более детального ознакомления [статью на habr](https://habr.com/ru/company/senetsy/blog/327456/).

Краткое описание:

Container LSP создаёт sub-LSPs основываясь на вводных параметрах + локальной статистике.
Например, есть номинальные 2 sub-LSPs в контейнере с min bw 1g, max bw 8g, так же указывается, что надо делать splitting \(добавление sub-LSP\(s\) при достижении и/или преодолении указанного параметра в splitting-bandwidth\) с функцией sampling, учитывая один из возможных [алгоритмов](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/sampling-edit-protocols-mpls-container-label-switched-path-splitting-merging.html) за период нормализации, например, при достижении и/или преодолении 5g sampling average aggregate.
При достижении и/или преодолении sampling average aggregate в течении периода нормализации добавится ещё sub-LSP\(s\)*n, с такими-же параметрами \(sub-LSPs наследуют настройки контейнера и во время нормализации происходит распределение bw между всеми sub-LSPs принадлежащими к этому контейнеру\), и будет участвовать в ECMP.

- LSP splitting - механизм разделения \(splitting\) LSP позволяет ingress маршрутизатору создавать новые sub-LSPs или повторно сигнализировать существующие LSP с различной полосой пропускания в контейнерном LSP, когда требование X помещается в контейнерный LSP. В текущей реализации ingress маршрутизатор пытается найти LSP, удовлетворяющий требованию X и другим ограничениям. Если путь не найден - либо LSP не сигнализируется, либо остается включенным, но со старой зарезервированной полосой пропускания. Между двумя событиями нормализации (расщепление или слияние) отдельные LSP могут повторно сигнализироваться с различной пропускной способностью из-за настроек auto-bw. Если контейнерный LSP не сконфигурирован с auto-bw, то sub-LSPs сигнализируются со статическим значением полосы пропускания, если настроено, в таком случае динамического разделения не происходит, так как отсутствует динамическая оценка совокупной полосы пропускания \(aggregate utilization\). Настройки разделения с определенным значением пропускной способности могут быть инициированы вручную.

  Дополнительно:
  - Между двумя событиями нормализации два LSP могут иметь разные полосы пропускания, на которые наложены ограничения auto-bw;
  - После того, как LSP разделены (или объединены), make-before-break использует совместное использование стиля с фиксированным фильтром (FF), если не настроен адаптивный параметр \(adaptive\). Однако, два разных LSP не выполняют совместное использование явного (SE) стиля для этой функции;
  - Когда LSP повторно сигнализируются с измененной пропускной способностью, некоторые из LSP могут не сигнализироваться успешно, что приводит к вариантам аварийного переключения.
- LSP merging - механизм слияния \(merging\) LSP позволяет ingress маршрутизатору удалять существующие sub-LSPs, это уменьшает количество сохраняемой информации о сети \(state\). Ingress маршрутизатор учитывает совокупную потребность и минимальное (или максимальное) количество LSP, и пересматривает количество sub-LSPs, которые должны быть активны на ingress маршрутизаторе. В результате периодически может происходить следующее при срабатывании таймера нормализации:
  - Повторная сигнализация некоторых существующих LSP с обновленной полосой пропускания;
  - Создание новых LSP;
  - Удаление некоторых из существующих LSP.

  Если контейнерный LSP не сконфигурирован с auto-bw, то sub-LSPs сигнализируются со статическим значением полосы пропускания, если настроено. Слияние LSP не происходит, потому что нет динамической оценки совокупной пропускной способности \(aggregate utilization\). Однако можно настроить ручной триггер для разделения и настройки с определенным значением пропускной способности.

  Дополнительно:
  - Номинальные LSP никогда не удаляются как часть слияния LSP;
  - Перед удалением sub-LSP, sub-LSP делается неактивным, поэтому трафик передается на другие sub-LSPs перед удалением sub-LSP. Это связано с тем, что RSVP отправляет PathTear перед удалением маршрутов и next hops из PFE;
  - Когда члены LSP повторно сигнализируются с измененной полосой пропускания, может случиться так, что некоторые LSP не будут успешно сигнализированы.

- Механизмы защиты LSP \(единовременно возможен только один вариант\):
  - Fast-reroute;
  - Link protection;
  - Node-link protection \(с возможностью деградации до Link protection\).

## Пример конфигурации

_Меня попросили для ясности описать почему использовались те или иные команды в конфигурации. Я опишу в комментариях к конфигурации часть команд не относящихся к данной статье. Для всего остального рекомендую воспользоваться [CLI Explorer](https://apps.juniper.net/cli-explorer/)_

```
groups {
    container-lsp {
        protocols {
            mpls {
                container-label-switched-path <*> {
                    label-switched-path-template {
                        lsp-template-backbone;
                    }
                    splitting-merging {
                        minimum-member-lsps 4;
                        maximum-member-lsps 12; ## если не задан maximum-member-lsps, то значение по умолчанию будет 64
                        splitting-bandwidth 5g;
                        merging-bandwidth 1g;
                        normalization {
                            normalize-interval 10800;
                            failover-normalization;
                            normalization-retry-duration 20;
                            normalization-retry-limits 3;
                        }
                        sampling {
                            use-average-aggregate;
                        }
                    }
                }
            }
        }
    }
    ospf-backbone {                     
        protocols {                   
            ospf {                      
                area 0.0.0.0 {          
                    interface <*> {     
                        interface-type p2p;
                        node-link-protection;
                        authentication {
                            md5 5 key "***"; ## SECRET-DATA
                        }
                        flood-reduction;
                        bfd-liveness-detection {
                            version automatic;
                            minimum-interval 50;
                            multiplier 3;
                        }
                    }
                }
            }
        }
    }
    rsvp-backbone {
        protocols {
            rsvp {
                interface <*> {
                    authentication-key "***"; ## SECRET-DATA
                    aggregate;
                    reliable;
                    subscription 85;
                    link-protection;
                }
            }
        }
    }
    interface-mpls {
        interfaces {
            <*> {
                unit 0 {
                    family mpls {
                        maximum-labels 5; ## Для EL, FRR и LDP over RSVP
                    }
                }
            }
        }
    }
}
chassis {
    network-services enhanced-ip; ## MUST
}
interfaces {
    xe-0/0/7 {
        apply-groups [ interface-mpls ];
        description "link with site2-p1";
        mtu 9200; ## depends on label stack
        hold-time up 5000 down 0; ## бывает, что обрыв ВОЛС происходит не сразу и интерфейс начинает флапать, чтобы избежать этого, используется hold-time up в совокупности с BER и/или OAM LFM \(которых нету в данной конфигурации в виду избыточности в рамках этой статьи\)
        unit 0 {
            family inet {
                address 10.10.10.10/31;
            }
        }
    }
}
routing-options {
  forwarding-table {
       export Load_balance; ## MUST
       ecmp-fast-reroute; ## полезная фича для уменьшения потерь во время выхода из группы ECMP NH\(s\)
   }
}
protocols {
    rsvp {
        apply-groups rsvp-backbone;
        node-hello;
        preemption aggressive;
        interface xe-0/0/7.0 {
            link-protection;
        }
    }
    mpls {
        statistics {
            file auto-bw size 10m;
            interval 60;
            auto-bandwidth;
            transit-statistics-polling;
            traffic-class-statistics;
        }
        optimize-adaptive-teardown {
            p2p;
        }
        smart-optimize-timer 180;
        label-switched-path lsp-template-backbone {
            template;
            ldp-tunneling;
            retry-timer 3;
            optimize-timer 3600;
            record;
            least-fill;
            adaptive;
            fast-reroute;
            auto-bandwidth {
                adjust-interval 7200;
                adjust-threshold 10;
                minimum-bandwidth 1k;
                maximum-bandwidth 8500000000;
                adjust-threshold-overflow-limit 5;
                adjust-threshold-underflow-limit 5;
                resignal-minimum-bandwidth;
            }
        }
        container-label-switched-path site2-p1 {
            apply-groups container-lsp;
            to 10.10.10.1;
        }
        interface xe-0/0/7.0;
    }
    ospf {
        spf-options {
            delay 50;
            holddown 5000;
            rapid-runs 3;
        }
        backup-spf-options {
            remote-backup-calculation;
            per-prefix-calculation all;
            node-link-degradation;
        }
        traffic-engineering; ## MUST
        reference-bandwidth 100g;
        lsa-refresh-interval 30;
        no-rfc-1583;
        area 0.0.0.0 {
            interface lo0.0;
            interface xe-0/0/7.0 {
                apply-groups ospf-backbone;
            }
        }
    }
    ldp { ## optional
        auto-targeted-session;
        track-igp-metric;
        mtu-discovery;
        deaggregate;
        interface lo0.0;
        session-protection;
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
```

## Ссылки
<b id="f1">1</b>. Dynamic Bandwidth Management Using Container LSP Overview: [https://www.juniper.net/documentation/en_US/junos/topics/concept/dynamic-bandwidth-management-overview.html](https://www.juniper.net/documentation/en_US/junos/topics/concept/dynamic-bandwidth-management-overview.html) [↩](#a1)<br/>
<b id="f2">2</b>. Multi-path Label Switched Paths Signaled Using RSVP-TE: [https://tools.ietf.org/html/draft-kompella-mpls-rsvp-ecmp-00](https://tools.ietf.org/html/draft-kompella-mpls-rsvp-ecmp-00) [↩](#a2)<br/>
<b id="f3">3</b>. Container LSP: [https://m.habr.com/ru/company/senetsy/blog/327456/](https://m.habr.com/ru/company/senetsy/blog/327456/) [↩](#a3)<br/>
<b id="f4">4</b>. bin-packing problem: [https://www.ietf.org/proceedings/82/slides/pce-11.pdf](https://www.ietf.org/proceedings/82/slides/pce-11.pdf) [↩](#a4)<br/>
