---
layout: post
title: "RPKI на Juniper MX"
tags: juniper rpki cloudflare ripe rtr validator
author: "slepwin"
---

RPKI на Juniper MX

> **Disclaimer**: Все описанное в статье претендует на капитанство.

## Установка и настройка RIPE Validator + RTR server

1. Установка из репозитория
```
sudo yum update
sudo yum-config-manager --add-repo https://ftp.ripe.net/tools/rpki/validator3/prod/centos7/ripencc-rpki-prod.repo
sudo yum install yum-utils
sudo yum install rpki-validator
sudo yum install rpki-rtr-server
sudo systemctl enable rpki-validator-3
sudo systemctl start rpki-validator-3
sudo systemctl enable rpki-rtr-server
sudo systemctl start rpki-rtr-server
```

2. Настройка конфигурации:
```
sudo nano /etc/rpki-validator-3/application.properties
server.address=0.0.0.0
```
```
sudo nano /etc/rpki-rtr-server/application.properties
server.address=0.0.0.0
rtr.server.address=0.0.0.0
rtr.server.port=8323
```
> **Примечание**: Для безопасности необходимо указать адреса интерфейсов на которых будут работать демоны, настроить фаервол а также NGINX для доступа по HTTP/HTTPS.

Путь к демонам (для информации):
```
/etc/systemd/system/rpki-validator-3.service
/etc/systemd/system/rpki-rtr-server.service
```
После смены настроек демона не забудьте:
```
sudo systemctl daemon-reload
```

3. Перезаруск демонов:
```
sudo systemctl restart rpki-validator-3
sudo systemctl restart rpki-rtr-server
```
RPKI Validator 3.1 доступен по http://x.x.x.x:8080
The RPKI-RTR Server доступен по tcp x.x.x.x:8323
Проверить API http://x.x.x.x:8080/swagger-ui.html

4. Установка ARIN TAL (остальные Trust Anchors built-in):
```
cd /tmp/
wget https://www.arin.net/resources/manage/rpki/arin-ripevalidator.tal
upload-tal.sh arin-ripevalidator.tal http://localhost:8080/
```

5. Проверка работы:
```
sudo systemctl status rpki-validator-3
sudo systemctl status rpki-rtr-server
netstat -tulpn
```

### Установка и настройка Cloudflare OctoRPKI + GoRTR server

1. Установка из RPM
```
sudo yum update
cd /opt/
sudo mkdir ./ringcentral
sudo wget https://github.com/cloudflare/cfrpki/releases/download/v1.1.4/octorpki-1.1.4-1.x86_64.rpm
sudo wget https://github.com/cloudflare/gortr/releases/download/v0.14.4/gortr-0.14.4-1.x86_64.rpm
sudo yum localinstall octorpki-1.1.4-1.x86_64.rpm
sudo yum localinstall gortr-0.14.4-1.x86_64.rpm
```

2. Настройка демонов:

- OctoRPKI
```
sudo nano /usr/lib/systemd/system/octorpki.service
```
```
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
```

- GoRTR
```
sudo nano /usr/lib/systemd/system/gortr.service
```
```
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
```

3. Перергружаем демоны в systemd:
```
sudo systemctl daemon-reload
```

4. Создаем папку для ключевой пары:
```
sudo mkdir -p /opt/cfrpki/keys/
cd /opt/cfrpki/keys/
```

5. Генерируем ключевую пару:
```
sudo openssl ecparam -genkey -name prime256v1 -noout -outform pem > private.pem
sudo openssl ec -in private.pem -pubout -outform pem > public.pem
```

6. Установка ARIN TAL (остальные Trust Anchors built-in):
```
sudo cd /usr/share/octorpki/tals
wget https://www.arin.net/resources/manage/rpki/arin-rfc7730.tal
```

7. Установка в автозагрузку и старт демонов:
```
sudo systemctl enable octorpki
sudo systemctl start octorpki
sudo systemctl enable gortr
sudo systemctl start gortr
```

8. Проверка работы:
```
sudo systemctl status octorpki
sudo systemctl status gortr
netstat -tulpn
```

### Настройка RPKI на Juniper MX:

> **Примечание**: x.x.x.x - RIPE Validator, y.y.y.y - Cloudflare Validator, a.a.a.a - MX Router 1, b.b.b.b - MX Router 2. В качестве примера выступает Stub AS.

1. Настройка RPKI сессий:
```
set routing-options validation group RPKI-SRV session x.x.x.x port 8323
set routing-options validation group RPKI-SRV session x.x.x.x local-address a.a.a.a
set routing-options validation group RPKI-SRV session y.y.y.y port 8323
set routing-options validation group RPKI-SRV session y.y.y.y local-address a.a.a.a
```

2. Настройка apply-path для PROTECT-RE:
```
set policy-options prefix-list RPKI-SRV apply-path "routing-options validation group <*> session <*>"
set policy-options prefix-list RPKI-LO apply-path "routing-options validation group <*> session <*> local-address <*>"
```

3. Настройка PROTECT-RE фильтра:
```
set firewall family inet filter PROTECT-RE term rpki-accept from source-prefix-list RPKI-SRV
set firewall family inet filter PROTECT-RE term rpki-accept from destination-prefix-list RPKI-LO
set firewall family inet filter PROTECT-RE term rpki-accept from protocol tcp
set firewall family inet filter PROTECT-RE term rpki-accept then count rpki-accept
set firewall family inet filter PROTECT-RE term rpki-accept then accept
```

4. Настройка комьюнити для validation state:
```
set policy-options community origin-validation-state-invalid members 0x4300:0.0.0.0:2
set policy-options community origin-validation-state-unknown members 0x4300:0.0.0.0:1
set policy-options community origin-validation-state-valid members 0x4300:0.0.0.0:0
```

5. Настройка политик импорта для EBGP соседей:
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

6. Настройка политик импорта для IBGP соседей:
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

6. Настройка политик экспорта для IBGP соседей:
```
set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET from protocol bgp
set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET then next-hop self
set policy-options policy-statement ACCEPT-FV-IBGP term FROM_INTERNET then accept
```

7. Настройка IBGP соседа/RR:
```
set protocols bgp group IBGP-RTR type internal
set protocols bgp group IBGP-RTR local-address a.a.a.a
set protocols bgp group IBGP-RTR import ACCEPT-RPKI-IBGP
set protocols bgp group IBGP-RTR export ACCEPT-FV-IBGP
set protocols bgp group IBGP-RTR neighbor b.b.b.b
```

8. Настройка EBGP соседа (uplink):
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

9. Проверка конфигурации:
```
show validation session detail
show validation database
show validation statistics
show route 1.0.0.0/24
```

## Ссылки
<b id="f1">1</b>. [Day One BGP Secure Routing
](https://www.juniper.net/documentation/en_US/day-one-books/DO_BGP_SecureRouting2.0.pdf) [↩](#a1)<br/>
<b id="f2">2</b>. [The Cloudflare Blog - RPKI](https://blog.cloudflare.com/rpki/) [↩](#a2)<br/>
<b id="f3">3</b>. [RPKI Documentation](https://rpki.readthedocs.io/en/latest/) [↩](#a3)<br/>

