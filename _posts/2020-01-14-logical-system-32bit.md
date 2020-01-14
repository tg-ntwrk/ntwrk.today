---
layout: post
title:  "Juniper в дефолте поднимет LS в 32bit режиме"
tags: juniper ls 32bit
author: "ipotech"
---

## Routing process daemon в Logical System в Junos Network Operating System

В cовременных Junos OS для Logical System по умолчанию запускается 32bit _rpd_ процесс. Соответственно, у этого процесса присутствует фактическое ограничение на адресацию более 3 и более GB RAM. Даже если сетевое оборудование Juniper Networks использует Routing Engine с 32 или более GB RAM в паре с 64bit Junos OS, во всех логических системах все равно будет запущен _rpd_ для 32bit архитектуры.

![32-64-bit](/images/32-64-bit-juniper.jpg)
В процессе экспериментов было обнаружено, что при 10M маршрутов _rpd_ в Logical System перезапускается при commit любой конфигурации.

Настройка 64bit rpd описана в официальной [документации](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/routing-edit-system-processes.html).

> **ВНИМАНИЕ**: При commit этих настроек произойдет перезапуск _rpd_ в Logical System.