---
layout: post
title:  "Дампы mpls пакетов при разных сценариях настройки между back-to-back маршрутизаторами Juniper MX"
tags: juniper mpls uhp
author: "ipotech"
---

Дампы MPLS пакетов при разных сценариях настройки между back-to-back маршрутизаторами Juniper MX

## Постановка вопроса
Имеем два Juniper MX, которые соединены друг с другом, то есть back-to-back (b2b).
Имеем транзитное оборудование, которое делает L2 связь между этими MX.
Транзитное оборудование имеет несколько ECMP и умеет смотреть только до MPLS заголовка для балансировки.

## Теория
При типе подключения b2b. Маршрутизатор Juniper не пушит транспортную MPLS метку для l3vpn, ни в режиме implicit-null, ни в режиме explicit-null. Только сервисную.

explicit-null по сути задумывался как раз для того чтобы оставить MPLS заголовок, например для CoS.

### explicit-null
Как я уже писал выше, не смотря на то, что в конфигурации указано:
```
show configuration protocols mpls explicit-null
explicit-null;
```
В дампе видно только сервисную метку:
![explicit-php](/images/explicit-b2b-php.png)

Выход - использовать UHP
### UHP
В режиме UHP транспортная метка есть всегда и она не нулевая.
Так же между Juniper MX можно поднять несколько параллельных UHP LSP, тем самым добавить вариативности.
Пример конфигурации:
```
show configuration protocols mpls
apply-groups mpls-lsp;
explicit-null;
optimize-timer 300;
label-switched-path lsp-to-a2-97_UHP1 {
    to 10.191.104.3;
    ultimate-hop-popping;
    primary to-a2-97;
}
label-switched-path lsp-to-a2-97_UHP2 {
    to 10.191.104.3;
    ultimate-hop-popping;
    primary to-a2-97;
}
label-switched-path lsp-to-a2-99_UHP1 {
    to 10.191.104.1;
    ultimate-hop-popping;
    primary to_A2-99;
}
label-switched-path lsp-to-a2-99_UHP2 {
    to 10.191.104.1;
    ultimate-hop-popping;
    primary to_A2-99;
}

path to-a2-97 {
    10.191.107.46 strict;
}
path to_A2-99 {
    10.191.107.125;
}
interface ae1.322;
interface ae5.352;
```

ниже дамп:

![explicit-uhp](/images/uhp-b2b.png)

### UHP + EL
Раз у нас есть две метки, то грех не вставить еще одну для еще большей вариативности, подумал я, но:
```
admin@mx80-all# show protocols mpls label-switched-path lsp-to-a2-97 | display set
set protocols mpls label-switched-path lsp-to-a2-97 to 10.191.107.46
set protocols mpls label-switched-path lsp-to-a2-97 ultimate-hop-popping
set protocols mpls label-switched-path lsp-to-a2-97 entropy-label
admin@mx80-all# commit check
[edit protocols mpls]
  'label-switched-path lsp-to-a2-97'
    entropy-label unsupported for UHP LSP
```
В сообществе высказали такую версию:
```
там есть два варианта как жить с el , один из них подразумевает что el должна быть попнута на PHP
и джун на данный момент поддерживает именно такой вариант
вероятно в будущем будет возможность с el на uhp
по сути , это проблема el , что трафик на PE приходит без el и как следствие не может использовать ее для балансировки. В этом плане fat pw будет лучше , т.к. метка будет сохраняться до конца ..
```

## Ссылки
1. Статья про UHP на сайте Juniper https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/mpls-ultimate-hop-popping-enabling.html
2. Включение explicit-null на сайт Juniper https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/explicit-null-edit-protocols-mpls.html
3. Различие explicit-null и implicit-null https://www.networkworld.com/article/2350466/understanding-mpls-explicit-and-implicit-null-labels.html
