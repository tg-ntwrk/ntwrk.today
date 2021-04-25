---
layout: post
title: "Build your own VPN node with Traefik v2, MTProto Proxy, WireGuard and BIRD 2.0 / Part 2"
tags: linux docker traefik telegram mtproto wireguard bird2
author: "github:freefd"
---

Today plenty of people already know what [VPN](https://en.wikipedia.org/wiki/Virtual_private_network) <sup id="a1">[1](#f1)</sup> is and why they need to use it. Many ISPs have the same knowledge and need to block VPNs in accordance with country laws. In this article we build uncommon installation for you own private VPN node.

## Server Side Configurations

Please see [Part 1](/2021/04/19/vpn-node-on-your-own-part-1.html) <sup id="a2">[2](#f2)</sup> for overall solution overview.

> *Note*: We will not attempt to build [SD-WAN](https://en.wikipedia.org/wiki/SD-WAN) <sup id="a3">[3](#f3)</sup> through this series of articles, nor will we implement any software controller for management plane. These articles describe topology contained only one VPN host communicating with two Linux-based [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment)s <sup id="a4">[4](#f4)</sup>. But this doesn't mean it cannot be scaled horizontally.

The [Docker Compose](https://docs.docker.com/compose/) <sup id="a5">[5](#f5)</sup> file docker-compose.yaml. Please see VPN box configurations in [repository](https://github.com/freefd/vpn_box/blob/main/server/docker-compose.yaml) <sup id="a6">[6](#f6)</sup> for more information:
```yaml
version: "2.4"

networks:
  1frontend:
     name: 1frontend
     driver: bridge
     ipam:
      config:
        - subnet: 192.168.100.0/24
          gateway: 192.168.100.1

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    command:
      - --entrypoints.http.address=:80/tcp
      - --entrypoints.https.address=:443/tcp
      - --entrypoints.wireguard.address=:443/udp
      - --providers.docker
      - --api
      - --log=true
      - --log.level=${TRAEFIK_LOG_LEVEL}
      - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
      - --certificatesresolvers.leresolver.acme.email=${TRAEFIK_ACME_MAIL}
      - --certificatesresolvers.leresolver.acme.storage=/acme.json
      - --certificatesresolvers.leresolver.acme.tlschallenge=true
    ports:
      - 80:80/tcp
      - 443:443/tcp
      - 443:443/udp
    networks:
      - 1frontend
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /docker/traefik/acme.json:/acme.json
      - /docker/traefik/htpasswd:/.htpasswd
    labels:
      # Dashboard
      - traefik.http.routers.traefik.rule=Host(`${TRAEFIK_DASHBOARD_HOST}`)
      - traefik.http.routers.traefik.service=api@internal
      - traefik.http.routers.traefik.tls.certresolver=leresolver
      - traefik.http.routers.traefik.entrypoints=https
      - traefik.http.routers.traefik.middlewares=authtraefik
      - traefik.http.middlewares.authtraefik.basicauth.usersfile=/.htpasswd
     
      # global redirect to https
      - traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)
      - traefik.http.routers.http-catchall.entrypoints=http
      - traefik.http.routers.http-catchall.middlewares=redirect-to-https

      # middleware redirect
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
      - traefik.docker.network=1frontend

  wireguard:
    container_name: wireguard
    build:
      context: wireguard
      dockerfile: /docker/wireguard/build/Dockerfile
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TIMEZONE}
    networks:
      - 1frontend
    volumes:
      - /docker/wireguard/periodic-config:/etc/periodic
      - /docker/wireguard/wireguard-config:/etc/wireguard
      - /docker/wireguard/bird-config:/etc/bird
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
      - net.ipv4.tcp_congestion_control=bbr
    labels:
      - traefik.enable=true
      - traefik.udp.routers.wireguard.entrypoints=wireguard
      - traefik.udp.services.wireguard.loadbalancer.server.port=51820
    restart: unless-stopped
    depends_on:
      - traefik

  mtg:
    image: nineseconds/mtg
    container_name: mtg
    networks:
      - 1frontend
    volumes:
      - /docker/mtg/mtg-config/config.toml:/config.toml
    labels:
      - traefik.enable=true
      - traefik.tcp.routers.mtg.entrypoints=https
      - traefik.tcp.routers.mtg.rule=HostSNI(`${MTG_TLS_HOST}`)
      - traefik.tcp.routers.mtg.tls.passthrough=true
      - traefik.tcp.services.mtg.loadbalancer.server.port=3128
    restart: unless-stopped
    depends_on:
      - traefik

  webhost:
    image: nginx
    container_name: webhost
    networks:
      - 1frontend
    volumes:
      - /docker/webhost/html:/usr/share/nginx/html:ro
    labels:
      - traefik.enable=true
      - traefik.http.routers.webhost.entrypoints=https
      - traefik.http.routers.webhost.rule=Host(`${WEBHOST_TLS_HOST}`)
      - traefik.http.routers.webhost.tls=true
      - traefik.http.routers.webhost.tls.certresolver=leresolver
    restart: unless-stopped
    depends_on:
      - traefik
```

It creates independent network named `1frontend` with [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) <sup id="a7">[7](#f7)</sup> `192.168.100.0/24`, then Traefik container will appear first using this network as default for connections to all other containers. There are defined several major things like endpoints:
 * 80/TCP for HTTP
 * 443/TCP for HTTPS
 * 443/UDP for [WireGuard](https://www.wireguard.com/) <sup id="a8">[8](#f8)</sup>

[ACME](https://tools.ietf.org/html/rfc8555) <sup id="a9">[9](#f9)</sup> settings to make [Let's Encrypt](https://letsencrypt.org/) <sup id="a10">[10](#f10)</sup> TLS certificates on-the-fly and [Basic Authentication](https://tools.ietf.org/html/rfc7617) <sup id="a11">[11](#f11)</sup> configuration to protect Traefik dashboard.

The WireGuard container comprising several special features:
 * WireGuard Server
 * [BIRD 2.0](https://bird.network.cz/) <sup id="a12">[12](#f12)</sup> daemon
 * Built-in [cron](https://en.wikipedia.org/wiki/Cron) <sup id="a13">[13](#f13)</sup>

Overall, target is the following directories structure:
```
root@host:~# tree -a /docker
/docker
├── docker-compose.yaml
├── .env
├── mtg
│   └── mtg-config
│       └── template.toml
├── traefik
│   ├── acme.json
│   └── htpasswd
├── webhost
│   └── html
│       ├── index.html
│       └── ntwrk.png
└── wireguard
    ├── bird-config
    │   ├── bird.conf
    │   ├── exclusions.conf
    │   ├── generated.conf
    │   └── manual.conf
    ├── build
    │   ├── Dockerfile
    │   └── supervisord.conf
    ├── periodic-config
    │   ├── 15min
    │   │   └── make_routes
    │   ├── daily
    │   ├── hourly
    │   ├── monthly
    │   └── weekly
    └── wireguard-config
        ├── bootstrap.sh
        ├── configurations
        │   ├── peer1.conf
        │   ├── peer2.conf
        │   ├── publickey-server.wg0
        │   └── server.conf
        └── templates
            ├── peer.wg0.conf
            └── server.wg0.conf
```

### Environment file .env
Environment file [.env](https://github.com/freefd/vpn_box/blob/main/server/.env) is used to keep variables for Docker Compose file.
```bash
# Common
PUID=1001
PGID=1001
UMASK_SET=0022
TIMEZONE=Europe/Moscow

# Container Specific
TRAEFIK_LOG_LEVEL=INFO
TRAEFIK_ACME_MAIL=acme@ntwrk.today
TRAEFIK_DASHBOARD_HOST=dashboard.ntwrk.today

MTG_TLS_HOST=duckduckgo.com

WEBHOST_TLS_HOST=webhost.ntwrk.today
```

### Traefik
Traefik Dashboard will be available at URI defined in `.env` file. The `htpasswd` file contains Basic Authentication credentials to protect Traefik Dashboard, the format is compatible with [htpasswd](https://doc.traefik.io/traefik/middlewares/basicauth/#general) <sup id="a14">[14](#f14)</sup> tool. In this guide we use `ntwrk:_ntVVrk$T0d4Y` that becomes:
```
root@host:~# cat /docker/traefik/htpasswd 
ntwrk:$apr1$24xr5mbt$YdPRnBWP1oUQWyLq/BfS4.
```
Do not forget to use [CAA](https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization) <sup id="a15">[15](#f15)</sup> record for domain with ACME mail:
```
root@host:~# host -t caa ntwrk.today
ntwrk.today has CAA record 0 issuewild "letsencrypt.org"
ntwrk.today has CAA record 0 issue "letsencrypt.org"
ntwrk.today has CAA record 0 iodef "mailto:acme@ntwrk.today"
```

### WireGuard Server
Let's drill down to WireGuard files and review the [Dockerfile](https://github.com/freefd/vpn_box/blob/main/server/wireguard/build/Dockerfile) first in build directory:
```bash
FROM alpine:edge

COPY build/supervisord.conf /etc/supervisord.conf
# You can get parser's source from https://github.com/unsacrificed/network-list-parser/
RUN echo 'https://dl-cdn.alpinelinux.org/alpine/edge/testing' >> /etc/apk/repositories && \
    apk --update --no-cache add git wireguard-tools bird py3-setuptools supervisor && \
    wget https://github.com/unsacrificed/network-list-parser/releases/download/v1.2/network-list-parser-linux-amd64-1.2.bin \
    -O /usr/local/bin/parser && \
    rm -f /etc/bird.conf && \
    chmod a+x /usr/local/bin/parser && \
    mkdir /etc/bird/

VOLUME /etc/periodic/
VOLUME /etc/bird
VOLUME /etc/wireguard/

EXPOSE 51820/udp

CMD ["supervisord","-c","/etc/supervisord.conf"]
```

The [Alpine Linux](https://alpinelinux.org/) <sup id="a16">[16](#f16)</sup> used to build this image, pushing supervisord.conf and downloading special [parser](https://github.com/unsacrificed/network-list-parser/) <sup id="a17">[17](#f17)</sup> binary, it will also contain
 * git
 * wireguard tools
 * bird
 * supervisor and py3-setuptools

artifacts, 3 volumes, exposed 51820/UDP port and, finally, [supervisor](http://supervisord.org/) <sup id="a18">[18](#f18)</sup> daemon. As it was mentioned earlier, our image carries 3 services defined in supervisord.conf:

```
[supervisord]
nodaemon=true
user=root

# One-shot generate WireGuard configuration
# and start WireGuard server
[program:wireguard]
command=sh /etc/wireguard/bootstrap.sh
startsecs=0
autostart=true
autorestart=false
stderr_logfile=/dev/stdout
stderr_logfile_maxbytes = 0
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes = 0

# Start BIRD 2.0 daemon
[program:bird]
command=bird -c /etc/bird/bird.conf
autorestart=true
stderr_logfile=/dev/stdout
stderr_logfile_maxbytes = 0
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes = 0

# Start Crond daemon
[program:crond] 
command=crond -l 2 -f
autostart=true
autorestart=true
stderr_logfile=/dev/stdout
stderr_logfile_maxbytes = 0
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes = 0
```
To create initial configuration for WireGuard it's possible to use ephemeral container with mounted volume from `configurations` directory:
```
root@host:~# docker run --rm -ti -v /docker/wireguard/wireguard-config/configurations/:/mnt alpine:edge ash
```
Add wireguard-tools package:
```
/ # apk --update add wireguard-tools
fetch https://dl-cdn.alpinelinux.org/alpine/edge/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/edge/community/x86_64/APKINDEX.tar.gz
(1/20) Installing wireguard-tools-wg (1.0.20210223-r0)
... omitted for brevity ...
(20/20) Installing wireguard-tools (1.0.20210223-r0)
Executing busybox-1.31.1-r21.trigger
OK: 14 MiB in 34 packages
```
Alter the umask temporarily to ensure that access is only restricted to the owner. Then run generation of Private and Public keys:
```
/ # umask 077
/ # wg genkey | tee /mnt/server.conf | wg pubkey > /mnt/publickey-server.wg0
/ # ls -la /mnt/
total 16
drwxr-xr-x    2 root     root          4096 Apr 18 15:39 .
drwxr-xr-x    1 root     root          4096 Apr 18 15:38 ..
-rw-------    1 root     root            45 Apr 18 15:39 publickey-server.wg0
-rw-------    1 root     root            45 Apr 18 15:39 server.conf
```
`exit` will auto-remove ephemeral container:
```
/ # exit
```
Check files exist in `configurations` directory on host:
```
root@host:~# ls /docker/wireguard/wireguard-config/configurations/
publickey-server.wg0  server.conf
root@host:~# cat /docker/wireguard/wireguard-config/configurations/server.conf
+AgasB8xc0Ip0upAp/MdC+YFaczM3VD5t4hZSoEek2c=
```
Modify [server.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/wireguard-config/configurations/server.conf) to make it compatible with simple templating engine used in our image:
```
root@host:~# sed -i '1 s/^/Server Configuration\n/' /docker/wireguard/wireguard-config/configurations/server.conf
root@host:~# echo 10.10.1.1/24 >> /docker/wireguard/wireguard-config/configurations/server.conf
root@host:~# cat /docker/wireguard/wireguard-config/configurations/server.conf
Server Configuration
+AgasB8xc0Ip0upAp/MdC+YFaczM3VD5t4hZSoEek2c=
10.10.1.1/24
```
The 1<sup>st</sup> line represents a comment, 2<sup>nd</sup> one is Private Key of WireGuard server, 3<sup>rd</sup> is for Address uses for VPN overlay on server side. Files [peer1.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/wireguard-config/configurations/peer1.conf) and [peer2.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/wireguard-config/configurations/peer2.conf) look exactly the same you expected, for example:
```
cpe1.ntwrk.today
tzIgY54n0Bx+1rIV2D/M8rbZyIxL4dTyYnto+J3U8Cg=
10.10.1.2/32, 192.168.101.0/24
```

```
cpe2.ntwrk.today
PhHXCXEos//WLpEisW4vBmwUuUOICiEInvDMjtnHTRs=
10.10.1.3/32, 192.168.102.0/24
```
The 1<sup>st</sup> line represents comment as well, 2<sup>nd</sup> is Public Key of the WireGuard client, 3<sup>rd</sup> is for AllowedIPs of client: the tunnel endpoint and the routed network behind it. Please recall topology diagram from [Part 1](/2021/04/19/vpn-node-on-your-own-part-1.html#solution-architecture) <sup id="a2">[2](#f2)</sup>.

During container startup the templating engine hidden by [bootstrap.sh](https://github.com/freefd/vpn_box/blob/main/server/wireguard/wireguard-config/bootstrap.sh) reads files in `configurations` directory and builds temporary `wg0.conf` for WireGuard server.

### BIRD 2.0
Now we're ready to review BIRD 2.0 configuration. Main configuration [bird.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/bird-config/bird.conf) file:
```
log syslog all;

ipv4 table master4;
ipv6 table master6;

protocol device { }

protocol kernel kernel4 {
    ipv4 {
        import none;
        export none;
    };
}

protocol kernel kernel6 {
    ipv6 {
        import none;
        export none;
    };
}

### Filters ###

protocol static static_bgp {
    ipv4;
    include "/etc/bird/manual.conf";
    include "/etc/bird/generated.conf";
}

### BGP Templates ###

template bgp branch_Peer {
    bfd;
    connect retry time 10;
    startup hold time 30;
    hold time 60;
    graceful restart;
    router id 10.10.1.1;
    local as 65001;

    ipv4 {
      import all;
      export all;
    };
}

template bgp roadwarrior_Peer {
    connect retry time 10;
    startup hold time 30;
    hold time 60;
    router id 10.10.1.1;
    local as 65001;

    ipv4 {
      import all;
      export all;
    };
}

### Branch Peers ###

protocol bgp branch_Router_BGP_1 from branch_Peer {
    neighbor 10.10.1.2 as 65002;
}

protocol bgp branch_Router_BGP_2 from branch_Peer {
    neighbor 10.10.1.3 as 65003;
}

protocol bfd branch_Router_BFD {
    neighbor 10.10.1.2;
    neighbor 10.10.1.3;
}

### Road Warriors ###

protocol bgp roadwarrior_BGP_1 from roadwarrior_Peer {
    neighbor 10.10.1.6 as 65006;
}
```
Templates for [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) <sup id="a19">[19](#f19)</sup> peers at branch premises and [road warrior](https://en.wikipedia.org/wiki/Road_warrior_(computing)) <sup id="a20">[20](#f20)</sup> BGP peers are defined there, it's simplify the manner on how peers can be declared. The main difference, road warrior peers do not contain [BFD](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) <sup id="a21">[21](#f21)</sup> protocol due to the poor quality of channels used by mobile clients. Also IPv4 table will be populated by static files with prefixes. The static file [manual.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/bird-config/manual.conf) filled manually for some specific time-invariant prefixes, for example:
```
### Linkedin
route 108.174.0.0/20 via "wg0";
route 185.63.144.0/22 via "wg0";
route 13.64.0.0/11 via "wg0";
route 13.96.0.0/13 via "wg0";
route 13.104.0.0/14 via "wg0";
```
The [generated.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/bird-config/generated.conf) file contains the same prefixes format but refreshes every 15 minutes by abovementioned [make_routes](https://github.com/freefd/vpn_box/blob/main/server/wireguard/periodic-config/15min/make_routes) script you noted in the directories structure:
```bash
#!/usr/bin/env sh

echo "Running make_routes script"
rm -rf /tmp/z-i
git clone --depth=1 https://github.com/zapret-info/z-i.git /tmp/z-i

# You can get parser's source from https://github.com/unsacrificed/network-list-parser/
echo "Generating prefixes"
parser -src-file /tmp/z-i/dump.csv -prefix 'route ' -suffix ' via "wg0";' 2>/dev/null > /etc/bird/generated.conf

echo "Excluding certain prefixes"
while read line;
do
    sed -i "/$line/d" /etc/bird/generated.conf;
done < /etc/bird/exclusions.conf

echo "Reloading BIRD"
birdc configure
rm -rf /tmp/z-i
```
Script creates a list of prefixes collected from [Roskomnadzor blacklists](https://eais.rkn.gov.ru/en/) <sup id="a22">[22](#f22)</sup> compiled into the [Register of Internet Addresses filtered in Russian Federation](https://github.com/zapret-info/z-i.git) <sup id="a23">[23](#f23)</sup> repository.

The [exclusions.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/bird-config/exclusions.conf) file is required to forcibly remove lines from [generated.conf](https://github.com/freefd/vpn_box/blob/main/server/wireguard/bird-config/generated.conf) file and exclude such prefixes to be advertised to VPN overlay. Format example:
```
185.165.123
185.71.67
```

### MTProto Proxy
[MTProto](https://core.telegram.org/mtproto) <sup id="a24">[24](#f24)</sup> proxy is a useful piece in our solution and we chose the [mtg](https://github.com/9seconds/mtg) <sup id="a25">[25](#f25)</sup> implementation. Generate new configuration with [Fake TLS](https://geekbrit.org/content/22070) <sup id="a26">[26](#f26)</sup> secret to simulate [duckduckgo.com](https://duckduckgo.com/) <sup id="a27">[27](#f27)</sup>:
```
root@host:~# mtg_secret=$(docker run --rm nineseconds/mtg generate-secret --hex duckduckgo.com); sed "s/__MTG_SECRET__/${mtg_secret}/" /docker/mtg/mtg-config/template.toml > /docker/mtg/mtg-config/config.toml
```
And put fronting domain name into `MTG_TLS_HOST` variable for [.env](https://github.com/freefd/vpn_box/blob/main/server/.env) file. Please pay attention on labels for `mtg` service in Docker Compose, you can find `traefik.tcp.routers.mtg.tls.passthrough=true` label which points Traefik not to terminate TLS session, this task will be done by `mtg` on its own, Traefik will route traffic based on [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) <sup id="a28">[28](#f28)</sup> field only.

### Webhost
Nothing to mention here, simple stateless Web server, you can modify it on your own.

## Run and Verify
It's time to run entire stack:
```
root@host:/docker# docker-compose up -d
Creating network "1frontend" with driver "bridge"
Pulling traefik (traefik:latest)...
... omitted for brevity ...
Status: Downloaded newer image for traefik:latest
Building wireguard
Step 1/8 : FROM alpine:edge
edge: Pulling from library/alpine
... omitted for brevity ...
Status: Downloaded newer image for alpine:edge
 ---> b0da5d0678e7
Step 2/8 : COPY build/supervisord.conf /etc/supervisord.conf
 ---> 380a8326000a
Step 3/8 : RUN echo 'https://dl-cdn.alpinelinux.org/alpine/edge/testing' >> /etc/apk/repositories &&     apk --update --no-cache add git wireguard-tools bird py3-setuptools supervisor &&     wget https://github.com/unsacrificed/network-list-parser/releases/download/v1.2/network-list-parser-linux-amd64-1.2.bin     -O /usr/local/bin/parser &&     rm -f /etc/bird.conf &&     chmod a+x /usr/local/bin/parser &&     mkdir /etc/bird/
 ---> Running in b98ba8d656b7
fetch https://dl-cdn.alpinelinux.org/alpine/edge/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/edge/community/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/edge/testing/x86_64/APKINDEX.tar.gz
(1/43) Installing ncurses-terminfo-base (6.2_p20210418-r0)
... omitted for brevity ...
(43/43) Installing wireguard-tools (1.0.20210315-r0)
Executing busybox-1.33.0-r2.trigger
Executing ca-certificates-20191127-r5.trigger
OK: 85 MiB in 57 packages
Connecting to github.com (140.82.114.3:443)
Connecting to github-releases.githubusercontent.com (185.199.109.154:443)
saving to '/usr/local/bin/parser'
parser                18% |******                          |  441k  0:00:04 ETA
parser               100% |********************************| 2340k  0:00:00 ETA
'/usr/local/bin/parser' saved
Removing intermediate container b98ba8d656b7
 ---> babc2d07dffd
Step 4/8 : VOLUME /etc/periodic/
 ---> Running in 46a53f07ca4b
Removing intermediate container 46a53f07ca4b
 ---> ca0fba4d8936
Step 5/8 : VOLUME /etc/bird
 ---> Running in 89eafeb9b69f
Removing intermediate container 89eafeb9b69f
 ---> 9a41e203a241
Step 6/8 : VOLUME /etc/wireguard/
 ---> Running in 51c922497c62
Removing intermediate container 51c922497c62
 ---> 1337741ffc34
Step 7/8 : EXPOSE 51820/udp
 ---> Running in 6f65d8d4000d
Removing intermediate container 6f65d8d4000d
 ---> 15e4d0e21c34
Step 8/8 : CMD ["supervisord","-c","/etc/supervisord.conf"]
 ---> Running in 7ff10c3acf59
Removing intermediate container 7ff10c3acf59
 ---> 5559378ca728

Successfully built 5559378ca728
Successfully tagged docker_wireguard:latest
WARNING: Image for service wireguard was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Pulling mtg (nineseconds/mtg:)...
... omitted for brevity ...
Status: Downloaded newer image for nineseconds/mtg:latest
Creating traefik ... done
Creating wireguard ... done
Creating mtg       ... done
```

In a few seconds after we can check Traefik status by login into Traefik dashboard and verify services are up and running:

![Traefik Dashboard](/images/2021-04-25-vpn-node-on-your-own-1.png)

This information can also be collected through [API](https://en.wikipedia.org/wiki/API) <sup id="a29">[29](#f29)</sup> we pre-enabled in the Traefik configuration. The entry points:
```
curl -su ntwrk:_ntVVrk$T0d4Y https://dashboard.ntwrk.today/api/entrypoints | jq '.[] | "\(.name) \(.address)"' -r | column -t
http       :80/tcp
https      :443/tcp
wireguard  :443/udp
```
And routers table where the columns are Entry Points, TLS, Rule, Name, Status:
```
for proto in http tcp udp; do curl -su ntwrk:_ntVVrk$T0d4Y https://dashboard.ntwrk.today/api/$proto/routers | jq '.[] | "\(.entryPoints[]) \(.tls) \(.rule) \(.name) \(.status)"' -r; done | column -t | sort -k4
http       null                           hostregexp(`{host:.+}`)        http-catchall@docker  enabled
https      {"passthrough":true}           HostSNI(`duckduckgo.com`)      mtg@docker            enabled
https      {"certResolver":"leresolver"}  Host(`dashboard.ntwrk.today`)  traefik@docker        enabled
https      {"certResolver":"leresolver"}  Host(`webhost.ntwrk.today`)    webhost@docker        enabled
wireguard  null                           null                           wireguard@docker      enabled
```
We must also verify status of WireGuard and BIRD inside the `wireguard` container. Since CPEs are still not configured, the WireGuard peers are shown as not connected yet:
```
root@host:~# docker exec wireguard wg show
interface: wg0
  public key: jyQ252wXhlkEpPdtHpC6DnTegKTxofapTENQyagJuhA=
  private key: (hidden)
  listening port: 51820

peer: tzIgY54n0Bx+1rIV2D/M8rbZyIxL4dTyYnto+J3U8Cg=
  allowed ips: 10.10.1.2/32, 192.168.101.0/24
  persistent keepalive: every 25 seconds

peer: PhHXCXEos//WLpEisW4vBmwUuUOICiEInvDMjtnHTRs=
  allowed ips: 10.10.1.3/32, 192.168.102.0/24
  persistent keepalive: every 25 seconds
```

Now checking the BIRD peers which are also not established due to peers are not reachable over the WireGuard tunnels:
```
root@host:~# docker exec wireguard birdc show proto
BIRD 2.0.8 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     14:29:02.682  
kernel4    Kernel     master4    up     14:29:02.682  
kernel6    Kernel     master6    up     14:29:02.682  
static_bgp Static     master4    up     14:29:02.682  
branch_Router_BGP_1 BGP        ---        start  14:29:02.682  Connect       Socket: Host is unreachable
branch_Router_BGP_2 BGP        ---        start  14:29:02.682  Connect       Socket: Host is unreachable
branch_Router_BFD BFD        ---        up     14:29:02.682  
roadwarrior_BGP_1 BGP        ---        start  14:29:02.682  Active        Socket: Host is unreachable
```

The last thing we need to check is that routing for networks behind CPEs was automatically added once `wg0` interface brought up:
```
root@main:~# docker exec wireguard ip route
default via 192.168.100.1 dev eth0
10.10.1.0/24 dev wg0 proto kernel scope link src 10.10.1.1
192.168.100.0/24 dev eth0 proto kernel scope link src 192.168.100.3
192.168.101.0/24 dev wg0 scope link
192.168.102.0/24 dev wg0 scope link
```
Part 3 explains CPEs and Device Endpoint configurations.

## References
<b id="f1">1</b>. [Virtual Private Network](https://en.wikipedia.org/wiki/Virtual_private_network) [↩](#a1)<br/>
<b id="f2">2</b>. [Build your own VPN node with Traefik v2, MTProto Proxy, WireGuard and BIRD 2.0 / Part 1](/2021/04/19/vpn-node-on-your-own-part-1.html) [↩](#a2)<br/>
<b id="f3">3</b>. [SD-WAN](https://en.wikipedia.org/wiki/SD-WAN) [↩](#a3)<br/>
<b id="f4">4</b>. [Customer Premises Equipment](https://en.wikipedia.org/wiki/Customer-premises_equipment) [↩](#a4)<br/>
<b id="f5">5</b>. [Docker Compose](https://docs.docker.com/compose/) [↩](#a5)<br/>
<b id="f6">6</b>. [VPN box repository](https://github.com/freefd/vpn_box/blob/main/server/docker-compose.yaml) [↩](#a6)<br/>
<b id="f7">7</b>. [Classless Inter-Domain Routing](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) [↩](#a7)<br/>
<b id="f8">8</b>. [WireGuard: fast, modern, secure VPN tunnel](https://www.wireguard.com/) [↩](#a8)<br/>
<b id="f9">9</b>. [Automatic Certificate Management Environment](https://tools.ietf.org/html/rfc8555) [↩](#a9)<br/>
<b id="f10">10</b>. [Let's Encrypt: free, automated, and open certificate authority](https://letsencrypt.org/) [↩](#a10)<br/>
<b id="f11">11</b>. [The Basic HTTP Authentication Scheme](https://tools.ietf.org/html/rfc7617) [↩](#a11)<br/>
<b id="f12">12</b>. [BIRD Internet Routing Daemon 2.0](https://bird.network.cz/) [↩](#a12)<br/>
<b id="f13">13</b>. [cron: time-based job scheduler in Unix-like OS](https://en.wikipedia.org/wiki/Cron) [↩](#a13)<br/>
<b id="f14">14</b>. [htpasswd: manage user files for Basic Authentication](https://doc.traefik.io/traefik/middlewares/basicauth/#general) [↩](#a14)<br/>
<b id="f15">15</b>. [DNS Certification Authority Authorization](https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization) [↩](#a15)<br/>
<b id="f16">16</b>. [Alpine Linux](https://alpinelinux.org/) [↩](#a16)<br/>
<b id="f17">17</b>. [network-list-parser: parse, normalize and aggregate list of IPv4 networks/addresses](https://github.com/unsacrificed/network-list-parser/) [↩](#a17)<br/>
<b id="f18">18</b>. [Supervisor: a process control system](http://supervisord.org/) [↩](#a18)<br/>
<b id="f19">19</b>. [Border Gateway Protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) [↩](#a19)<br/>
<b id="f20">20</b>. [Road Warrior](https://en.wikipedia.org/wiki/Road_warrior_(computing)) [↩](#a20)<br/>
<b id="f21">21</b>. [Bidirectional Forwarding Detection](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) [↩](#a21)<br/>
<b id="f22">22</b>. [Unified register of resources which are forbidden in the Russian Federation](https://eais.rkn.gov.ru/en/) [↩](#a22)<br/>
<b id="f23">23</b>. [Register of Internet Addresses filtered in Russian Federation](https://github.com/zapret-info/z-i.git) [↩](#a23)<br/>
<b id="f24">24</b>. [MTProto Mobile Protocol](https://core.telegram.org/mtproto) [↩](#a24)<br/>
<b id="f25">25</b>. [mtg: MTProto proxy for Telegram](https://github.com/9seconds/mtg) [↩](#a25)<br/>
<b id="f26">26</b>. [MTProto Fake TLS](https://geekbrit.org/content/22070) [↩](#a26)<br/>
<b id="f27">27</b>. [DuckDuckGo Internet Search Engine](https://duckduckgo.com/) [↩](#a27)<br/>
<b id="f28">28</b>. [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) [↩](#a28)<br/>
<b id="f29">29</b>. [Application Programming Interface](https://en.wikipedia.org/wiki/API) [↩](#a29)