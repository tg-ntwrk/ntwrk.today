---
layout: post
title: "RPKI инфраструктура: RIPE validator и RPKI-to-Router Protocol сервер для Juniper MX"
tags: juniper rpki cloudflare ripe gortr validator
author: "slepwin"
---

В статье описывается внедрение [RPKI](https://www.ripe.net/manage-ips-and-asns/resource-management/certification/what-is-rpki) <sup id="a1">[1](#f1)</sup> инфраструктуры на примере двух [RPKI Validator](https://www.ripe.net/manage-ips-and-asns/resource-management/certification/tools-and-resources) <sup id="a2">[2](#f2)</sup> и [RTR Server](https://rpki.readthedocs.io/en/latest/rpkivalidator3/#rpki-rtr-server) <sup id="a3">[3](#f3)</sup> от [RIPE NCC](https://www.ripe.net/) <sup id="a4">[4](#f4)</sup> и [Cloudflare](https://www.cloudflare.com/) <sup id="a5">[5](#f5)</sup> , а также соответствующая конфигурация Junos OS для Juniper MX.

## Сетевая топология

![RPKI-network-diagram](/images/2020-04-13-rpki-network-diagram.png)

## Установка и настройка RIPE Validator + RTR Server

1. Установка из репозитория [RIPE NCC](https://www.ripe.net/) <sup id="a4">[4](#f4)</sup> для дистрибутивов на базе [RHEL](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) <sup id="a6">[6](#f6)</sup>:
    ```bash
    sudo yum update
    # Adding RPM repo
    sudo yum-config-manager --add-repo https://ftp.ripe.net/tools/rpki/validator3/prod/centos7/ripencc-rpki-prod.repo

    # Install necessary packages
    sudo yum install yum-utils rpki-validator rpki-rtr-server

    # Enable services and run them
    sudo systemctl enable rpki-validator-3 rpki-rtr-server
    sudo systemctl start rpki-validator-3 rpki-rtr-server

    # Check both services are running
    sudo systemctl is-active rpki-validator-3 rpki-rtr-server
    ```

1. Базовая настройка конфигурации RPKI валидатора:
    ```bash
    # Edit RPKI validator configuration
    sudo nano /etc/rpki-validator-3/application.properties
    server.address=0.0.0.0
    ```

    Базовая настройка конфигурации RTR сервера:
    ```bash
    # Edit RTR server configuration
    sudo nano /etc/rpki-rtr-server/application.properties
    server.address=0.0.0.0
    rtr.server.address=0.0.0.0
    rtr.server.port=8323
    ```

    > **Примечание**: Для безопасности необходимо указать IP-адреса интерфейсов, на которых будут работать сервисы, настроить фаервол, а также реверс-прокси для доступа по HTTP/HTTPS.

1. Перезапуск сервисов:
    ```bash
    # Restart both services
    sudo systemctl restart rpki-validator-3 rpki-rtr-server
    ```

    * RPKI Validator 3.1 должен быть доступен по [http://x.x.x.x:8080/](http://x.x.x.x:8080/)
    * Проверить RPKI Validator API можно по [http://x.x.x.x:8080/swagger-ui.html](http://x.x.x.x:8080/swagger-ui.html)
    * RTR Server должен быть доступен по tcp x.x.x.x:8323

1. Установка ARIN TAL (остальные Trust Anchors встроены):
    ```bash
    # Download file to temporary directory
    wget https://www.arin.net/resources/manage/rpki/arin-ripevalidator.tal -O /tmp/arin-ripevalidator.tal
    # Install ARIN TAL
    upload-tal.sh /tmp/arin-ripevalidator.tal http://localhost:8080/
    # Remove previously downloaded file
    rm /tmp/arin-ripevalidator.tal
    ```

1. Проверка работоспособности сервисов:
    ```bash
    # Check status for both services
    sudo systemctl status rpki-validator-3 rpki-rtr-server
    sudo ss -tulpn | egrep '8080|8323'
    ```

### Установка и настройка Cloudflare OctoRPKI и GoRTR сервера

1. Установка [OctoRPKI](https://github.com/cloudflare/cfrpki) <sup id="a7">[7](#f7)</sup> и [GoRTR](https://github.com/cloudflare/gortr) <sup id="a8">[8](#f8)</sup> для дистрибутивов на базе RHEL
    ```bash
    # Install curl and jq as they are support tools
    sudo yum update
    sudo yum install curl jq
    # Install latest RPM packages for OctoRPKI and GoRTR
    sudo yum install $(for repo in cfrpki gortr; do curl -s https://api.github.com/repos/cloudflare/$repo/releases | jq 'first(.[].assets[] | select(.name | contains("rpm"))) | .browser_download_url' -r; done)
    ```
    Где вложенная команда:
    ```bash
    for repo in cfrpki gortr; do curl -s https://api.github.com/repos/cloudflare/$repo/releases | jq 'first(.[].assets[] | select(.name | contains("rpm"))) | .browser_download_url' -r; done
    ```

    Выбирает последние доступные версии RPM пакетов из релизов репозиториев [OctoRPKI](https://github.com/cloudflare/cfrpki) <sup id="a7">[7](#f7)</sup> и [GoRTR](https://github.com/cloudflare/gortr) <sup id="a8">[8](#f8)</sup>.

1. Настройка обоих сервисов:

    Создание файла сервиса systemd для [OctoRPKI](https://github.com/cloudflare/cfrpki) <sup id="a7">[7](#f7)</sup>:
    ```bash
    # Create OctoRPKI systemd service file
    sudo cat <<EOF > /etc/systemd/system/octorpki.service
    [Unit]
    Description=OctoRPKI
    After=network.target

    [Service]
    Type=simple
    EnvironmentFile=/etc/default/octorpki
    WorkingDirectory=/usr/share/octorpki
    ExecStart=/usr/bin/octorpki -mode server -output.sign.key /opt/cfrpki/keys/private.pem
    Restart=always
    RestartSec=60

    StandardOutput=syslog
    StandardError=syslog
    SyslogIdentifier=cf-octorpki

    [Install]
    WantedBy=multi-user.target

    EOF
    ```

    Создание файла сервиса systemd для [GoRTR](https://github.com/cloudflare/gortr) <sup id="a8">[8](#f8)</sup>:
    ```bash
    # Create GoRTR systemd service file
    sudo cat <<EOF > /etc/systemd/system/gortr.service
    [Unit]
    Description=GoRTR
    After=octorpki.service

    [Service]
    Type=simple
    EnvironmentFile=/etc/default/gortr
    WorkingDirectory=/usr/share/gortr
    ExecStartPre=/bin/sleep 30
    ExecStart=/usr/bin/gortr -bind :8323 -verify.key /opt/cfrpki/keys/public.pem -cache http://localhost:8080/output.json -metrics.addr :8081 -refresh=120
    Restart=always
    RestartSec=60

    StandardOutput=syslog
    StandardError=syslog
    SyslogIdentifier=cf-gortr

    [Install]
    WantedBy=multi-user.target

    EOF
    ```

1. Перезагрузка systemd для обнаружения новых сервисов:
    ```bash
    # Reload systemctl daemon
    sudo systemctl daemon-reload
    ```

1. Генерация пары ключей для [OctoRPKI](https://github.com/cloudflare/cfrpki) <sup id="a7">[7](#f7)</sup>:
    ```bash
    # Generate keypair for OctoRPKI
    sudo mkdir -p /opt/cfrpki/keys/
    sudo openssl ecparam -genkey -name prime256v1 -noout -outform pem > /opt/cfrpki/keys/private.pem
    sudo openssl ec -in /opt/cfrpki/keys/private.pem -pubout -outform pem > /opt/cfrpki/keys/public.pem
    ```

1. Установка [ARIN TAL](https://www.arin.net/resources/manage/rpki/tal/) <sup id="a9">[9](#f9)</sup> (остальные Trust Anchors встроены):
    ```bash
    # Download ARIN TAL
    sudo wget https://www.arin.net/resources/manage/rpki/arin-rfc7730.tal -O /usr/share/octorpki/tals/arin-rfc7730.tal
    ```

1. Добавление в загрузку при старте операционной системы и запуск сервисов:
    ```bash
    # Enable services and run them
    sudo systemctl enable octorpki gortr
    sudo systemctl start octorpki gortr
    # Check both services are running
    sudo systemctl is-active octorpki gortr
    ```

1. Проверка работоспособности сервисов:
    ```bash
    # Check status for both services
    sudo systemctl status octorpki gortr
    sudo ss -tulpn | egrep '8080|8323'
    ```

### Настройка RPKI на Juniper MX:

В качестве примера выступает Stub AS.
    
* x.x.x.x - RIPE RTR server 
* y.y.y.y - Cloudflare RTR server
* a.a.a.a - Loopback IP on MX Router 1
* b.b.b.b - Loopback IP on MX Router 2/RR
* c.c.c.c - IP from Peering network MX Router 
* d.d.d.d - IP from Peering network on Uplink Router (External AS)
* YYYY - AS number on Uplink Router (External AS)

1. Конфигурация RPKI сессий:
    ```
    set routing-options validation group RPKI-SRV session x.x.x.x port 8323
    set routing-options validation group RPKI-SRV session x.x.x.x local-address a.a.a.a
    set routing-options validation group RPKI-SRV session y.y.y.y port 8323
    set routing-options validation group RPKI-SRV session y.y.y.y local-address a.a.a.a
    ```

1. Конфигурация apply-path для PROTECT-RE фильтра:
    ```
    set policy-options prefix-list RPKI-SRV apply-path "routing-options validation group <*> session <*>"
    set policy-options prefix-list RPKI-LO apply-path "routing-options validation group <*> session <*> local-address <*>"
    ```

1. Конфигурация PROTECT-RE фильтра:
    ```
    set firewall family inet filter PROTECT-RE term rpki-accept from source-prefix-list RPKI-SRV
    set firewall family inet filter PROTECT-RE term rpki-accept from destination-prefix-list RPKI-LO
    set firewall family inet filter PROTECT-RE term rpki-accept from protocol tcp
    set firewall family inet filter PROTECT-RE term rpki-accept then count rpki-accept
    set firewall family inet filter PROTECT-RE term rpki-accept then accept
    ```

1. Конфигурация комьюнити для validation state:
    ```
    set policy-options community origin-validation-state-invalid members 0x4300:0.0.0.0:2
    set policy-options community origin-validation-state-unknown members 0x4300:0.0.0.0:1
    set policy-options community origin-validation-state-valid members 0x4300:0.0.0.0:0
    ```

1. Конфигурация политик импорта для EBGP соседей:
    > **Примечание**: Используйте одинаковые значения local-preference для valid и unknown маршрутов при первичной имплементации RPKI.

    ```
    set policy-options policy-statement ACCEPT-FV-RPKI term valid from protocol bgp
    set policy-options policy-statement ACCEPT-FV-RPKI term valid from validation-database valid
    set policy-options policy-statement ACCEPT-FV-RPKI term valid from route-filter 0.0.0.0/0 upto /24
    set policy-options policy-statement ACCEPT-FV-RPKI term valid then local-preference 110
    set policy-options policy-statement ACCEPT-FV-RPKI term valid then validation-state valid
    set policy-options policy-statement ACCEPT-FV-RPKI term valid then community add origin-validation-state-valid
    set policy-options policy-statement ACCEPT-FV-RPKI term valid then accept
    set policy-options policy-statement ACCEPT-FV-RPKI term invalid from protocol bgp
    set policy-options policy-statement ACCEPT-FV-RPKI term invalid from validation-database invalid
    set policy-options policy-statement ACCEPT-FV-RPKI term invalid then validation-state invalid
    set policy-options policy-statement ACCEPT-FV-RPKI term invalid then community add origin-validation-state-invalid
    set policy-options policy-statement ACCEPT-FV-RPKI term invalid then reject
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown from protocol bgp
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown from route-filter 0.0.0.0/0 upto /24
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown then local-preference 100
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown then validation-state unknown
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown then community add origin-validation-state-unknown
    set policy-options policy-statement ACCEPT-FV-RPKI term unknown then accept
    ```

1. Конфигурация политик импорта для IBGP соседей:
    ```
    set policy-options policy-statement ACCEPT-RPKI-IBGP term valid from community origin-validation-state-valid
    set policy-options policy-statement ACCEPT-RPKI-IBGP term valid then validation-state valid
    set policy-options policy-statement ACCEPT-RPKI-IBGP term valid then local-preference 110
    set policy-options policy-statement ACCEPT-RPKI-IBGP term invalid from community origin-validation-state-invalid
    set policy-options policy-statement ACCEPT-RPKI-IBGP term invalid then validation-state invalid
    set policy-options policy-statement ACCEPT-RPKI-IBGP term unknown from community origin-validation-state-unknown
    set policy-options policy-statement ACCEPT-RPKI-IBGP term unknown then validation-state unknown
    set policy-options policy-statement ACCEPT-RPKI-IBGP term unknown then local-preference 100
    ```

1. Конфигурация политик экспорта для IBGP соседей:
    ```
    set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET from protocol bgp
    set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET then next-hop self
    set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET then accept
    ```

1. Конфигурация IBGP соседа/RR:
    ```
    set protocols bgp group IBGP-RTR type internal
    set protocols bgp group IBGP-RTR local-address a.a.a.a
    set protocols bgp group IBGP-RTR import ACCEPT-RPKI-IBGP
    set protocols bgp group IBGP-RTR export ACCEPT-FV-IBGP
    set protocols bgp group IBGP-RTR neighbor b.b.b.b
    ```

1. Конфигурация EBGP соседа (uplink):
    ```
    set protocols bgp group EBGP_UPLINK_1 type external
    set protocols bgp group EBGP_UPLINK_1 local-address c.c.c.c
    set protocols bgp group EBGP_UPLINK_1 hold-time 360
    set protocols bgp group EBGP_UPLINK_1 keep none
    set protocols bgp group EBGP_UPLINK_1 log-updown
    set protocols bgp group EBGP_UPLINK_1 import REJECT-BOGON-ASNS
    set protocols bgp group EBGP_UPLINK_1 import REJECT-BOGONS
    set protocols bgp group EBGP_UPLINK_1 import REJECT-LOCAL-AS
    set protocols bgp group EBGP_UPLINK_1 import REJECT-LOCAL
    set protocols bgp group EBGP_UPLINK_1 import ACCEPT-FV-RPKI
    set protocols bgp group EBGP_UPLINK_1 import NONE
    set protocols bgp group EBGP_UPLINK_1 export ACCEPT_EXTERNAL
    set protocols bgp group EBGP_UPLINK_1 export NONE
    set protocols bgp group EBGP_UPLINK_1 remove-private
    set protocols bgp group EBGP_UPLINK_1 peer-as YYYY
    set protocols bgp group EBGP_UPLINK_1 graceful-restart
    set protocols bgp group EBGP_UPLINK_1 neighbor d.d.d.d
    ```

1. Проверка конфигурации:
    ```
    show validation session detail
    show validation database
    show validation statistics
    show route 1.0.0.0/24
    ```

## Ссылки
<b id="f1">1</b>. [What is RPKI?](https://www.ripe.net/manage-ips-and-asns/resource-management/certification/what-is-rpki) [↩](#a1)<br/>
<b id="f2">2</b>. [RPKI validator](https://www.ripe.net/manage-ips-and-asns/resource-management/certification/tools-and-resources) [↩](#a2)<br/>
<b id="f3">3</b>. [RPKI-to-Router protocol server](https://rpki.readthedocs.io/en/latest/rpkivalidator3/#rpki-rtr-server) [↩](#a3)<br/>
<b id="f4">4</b>. [RIPE Network Coordination Centre](https://www.ripe.net/) [↩](#a4)<br/>
<b id="f5">5</b>. [Cloudflare](https://www.cloudflare.com/) [↩](#a5)<br/>
<b id="f6">6</b>. [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) [↩](#a6)<br/>
<b id="f7">7</b>. [OctoRPKI standalone RPKI validator](https://github.com/cloudflare/cfrpki) [↩](#a7)<br/>
<b id="f8">8</b>. [GoRTR RPKI to Router protocol implementation](https://github.com/cloudflare/gortr) [↩](#a8)<br/>
<b id="f9">9</b>. [ARIN's Trust Anchor Locator](https://www.arin.net/resources/manage/rpki/tal/) [↩](#a9)<br/>
<b id="f10">10</b>. [The Cloudflare Blog - RPKI](https://blog.cloudflare.com/rpki/)<br/>
<b id="f11">11</b>. [RPKI Documentation](https://rpki.readthedocs.io/en/latest/)<br/>
<b id="f12">12</b>. [Day One BGP Secure Routing](https://www.juniper.net/documentation/en_US/day-one-books/DO_BGP_SecureRouting2.0.pdf)<br/>

