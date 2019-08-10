---
layout: post
title:  "Дампы MPLS пакетов при разных сценариях настройки между Back-to-Back маршрутизаторами Juniper MX"
tags: juniper mpls uhp
author: "ipotech"
---

Дампы MPLS пакетов при разных сценариях настройки между Back-to-Back маршрутизаторами Juniper MX

## Постановка вопроса
Два Juniper MX соединены друг с другом, то есть Back-to-Back (b2b), транзитное оборудование организует L2-связность между ними, имеет несколько ECMP и умеет обрабатывать пакеты только до MPLS заголовка для балансировки.

## Теория
При типе подключения b2b маршрутизатор Juniper не "пушит" транспортную MPLS метку для L3VPN, ни в режиме Implicit Null, ни в режиме Explicit Null. Только сервисную. Explicit Null задумывался как раз для того, чтобы оставить MPLS заголовок, например для CoS.

### Explicit Null
Несмотря на то, что в конфигурации указано:
```
show configuration protocols mpls explicit-null
explicit-null;
```
В дампе видно только сервисную метку:
![explicit-php](/images/explicit-b2b-php.png)

### UHP
В режиме UHP транспортная метка есть всегда и она не нулевая. Также между Juniper MX можно организовать несколько параллельных UHP LSP, добавив вариативности.

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

Дамп:

![explicit-uhp](/images/uhp-b2b.png)

### UHP + EL
Попытка добавить ещё одну метку для большей вариативности заканчивается ошибкой валидации:
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
Сообществе высказывает следующую версию:
```
Eсть два варианта, как жить с EL. Один из них подразумевает, что EL должна быть "попнута" на PHP,
и Juniper на данный момент поддерживает именно такой вариант.
Вероятно, в будущем будет возможность использовать EL c UHP.
По сути, это проблема EL, трафик на PE приходит без EL и,как следствие, не может использовать её для балансировки. В этом плане FAT PW будет лучше, так как метка будет сохраняться до конца.
```

## Ссылки
1. Статья про UHP на сайте Juniper https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/mpls-ultimate-hop-popping-enabling.html
2. Включение explicit-null на сайт Juniper https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/explicit-null-edit-protocols-mpls.html
3. Различие explicit-null и implicit-null https://www.networkworld.com/article/2350466/understanding-mpls-explicit-and-implicit-null-labels.html
