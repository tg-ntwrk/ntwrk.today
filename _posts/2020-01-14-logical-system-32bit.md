---
layout: post
title:  "32-bit routing process daemon в Logical System в Junos Network Operating System"
tags: juniper ls 32bit
author: "ipotech"
---

В cовременных Junos OS для Logical System по умолчанию запускается 32-bit _rpd_ процесс. Соответственно, у этого процесса присутствует фактическое ограничение на адресацию более 3 и более GB RAM. Даже если сетевое оборудование Juniper Networks использует Routing Engine с 32 или более GB RAM в паре с 64-bit Junos OS, во всех логических системах все равно будет запущен _rpd_ для 32-bit архитектуры.

![32-64-bit]({{ site.baseurl }}/images/32-64-bit-juniper.jpg)
В процессе экспериментов было обнаружено, что при 10M маршрутов _rpd_ в Logical System перезапускается при commit любой конфигурации.

Настройка 64-bit rpd описана в официальной [документации](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/routing-edit-system-processes.html).

> **ВНИМАНИЕ**: При commit этих настроек произойдет перезапуск _rpd_ в Logical System.