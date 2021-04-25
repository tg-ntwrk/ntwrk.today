---
layout: post
title: "Build your own VPN node with Traefik v2, MTProto Proxy, WireGuard and BIRD 2.0 / Part 1"
tags: linux docker traefik telegram mtproto wireguard bird
author: "github:freefd"
---

Today plenty of people already know what [VPN](https://en.wikipedia.org/wiki/Virtual_private_network) <sup id="a1">[1](#f1)</sup> is and why they need to use it. So many governments have joined Internet censorship, new examples appear every year all over the world. In this article we build uncommon installation for you own private VPN node.


## Solution Architecture
Our router is represented by [Traefik v2](https://doc.traefik.io/traefik/) <sup id="a2">[2](#f2)</sup> reverse proxy, it hides [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) <sup id="a3">[3](#f3)</sup> routed [MTProto](https://core.telegram.org/mtproto) <sup id="a4">[4](#f4)</sup> proxy in [Fake TLS](https://geekbrit.org/content/22070) <sup id="a5">[5](#f5)</sup> mode and [WireGuard](https://www.wireguard.com/) <sup id="a6">[6](#f6)</sup> with [BIRD 2.0](https://bird.network.cz/) <sup id="a7">[7](#f7)</sup>.

> *Note*: We will not attempt to build [SD-WAN](https://en.wikipedia.org/wiki/SD-WAN) <sup id="a8">[8](#f8)</sup> through this series of articles, nor will we implement any software controller for management plane. These articles describe topology contained only one VPN host communicating with two Linux-based [CPE](https://en.wikipedia.org/wiki/Customer-premises_equipment)s <sup id="a9">[9](#f9)</sup>. But this doesn't mean it cannot be scaled horizontally.

As we intend to make our VPN node stable and reachable as much as possible, we decided to use TCP and UDP ports to mimic to existing everyday applications:
* TCP/443 for MTProto
* UDP/443 for WireGuard

But please bear in mind, the [WireGuard does not focus on obfuscation](https://www.WireGuard.com/known-limitations/#deep-packet-inspection) <sup id="a10">[10](#f10)</sup> and does not try to hide itself from firewalls with [DPI](https://en.wikipedia.org/wiki/Deep_packet_inspection) <sup id="a11">[11](#f11)</sup> engine. We also will not try to hide WireGuard from ISP's backbone, but just select UDP/443 port that is used by [QUIC](https://en.wikipedia.org/wiki/QUIC) <sup id="a12">[12](#f12)</sup> protocol and is usually not blocked or inspected due to complex implementation for such a huge amount of traffic.

Overall diagram:

![Solution Architecture](/images/2021-04-19-vpn-node-on-your-own-1.png)

## Server Side Platform Overview
The server side platform doesn't matter as we use containerization. For this article use [Ubuntu Server 20.04 LTS](https://releases.ubuntu.com/20.04/) <sup id="a13">[13](#f13)</sup> as platform and [Docker](https://www.docker.com/) <sup id="a14">[14](#f14)</sup> with [Docker Compose](https://docs.docker.com/compose/) <sup id="a15">[15](#f15)</sup> for simple containers orchestration. Let me describe components one by one.

### Docker and Docker Compose
Docker has been installed from standard system repositories along with Docker Compose. The installation will contain four services: BIRD 2.0, WireGuard and NAT will be packed into single container with [supervisord](http://supervisord.org/) <sup id="a16">[16](#f16)</sup> to run multiple services, Traefik v2, MTProto Proxy and Simple HTTP service will run as independent containers. Docker Compose helps us to orchestrate microservices topology, Compose 2.4 specification will be used as there we still able to [create networks with names and gateways](https://github.com/docker/compose/issues/6569) <sup id="a17">[17](#f17)</sup>.

Of course, we would love to use [Podman](https://podman.io/) <sup id="a18">[18](#f18)</sup> instead of Docker, however, at the time of writing, the Traefik v2 is still [not fully compatible](https://github.com/traefik/traefik/issues/5730) <sup id="a19">[19](#f19)</sup> with Podman.

### Traefik v2
Traefik v2 is an open source edge router that provides publishing of services, it receives requests and finds out which components are responsible for handling them. This article is not intended to introduce you to Traefik v2 configuration, please check the official documentation.

### WireGuard
WireGuard is a free and open source software application and communication protocol that implements VPN techniques to create secure peer-to-peer connections in routed or bridged configurations. In charge of tunneling private traffic.

### BIRD 2.0
BIRD 2.0 is an open source implementation for routing Internet Protocol packets on Unix-like operating systems. It will be responsible for [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) <sup id="a20">[20](#f20)</sup> and [BFD](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) <sup id="a21">[21](#f21)</sup> communications. It'll push selected network prefixes toward client side while BFD is used to quickly shutdown BGP session when remote client stops responding to BFD probes.

### MTProto Proxy
MTProto proxy was created to bypass many kinds of government firewalls that block access to Telegram. The Fake TLS mode is a method that makes proxy traffic look like normal TLS. Therefore, it is more difficult to detect such a proxy, and therefore block Telegram too. Protects our connection to Telegram servers making remote breakout.

### HTTP Server
A simple HTTP server for holding any additional files which makes our installation look like HTTPS server.

## CPE Side Platform Overview
CPE platform may vary across *nix family, [OpenWrt 19.07](https://openwrt.org/) <sup id="a22">[22](#f22)</sup> was chosen for this article with additional packages installed:
* bird2, bird2c, bird2cl
* wireguard, wireguard-tools, luci-app-wireguard, luci-proto-wireguard, kmod-wireguard
* json4lua, luasec
* ca-certificates, openssh-server, openssh-client, openssh-keygen, shadow-common, shadow-useradd, shadow-userdel, unzip, zip, tcpdump, ddns-scripts, ddns-scripts-services, sudo

BIRD 2.0 works in pair with WireGuard client to create automated routing scheme in the following sequence:

![CPE Boot Up Sequence](/images/2021-04-19-vpn-node-on-your-own-2.png)

[Part 2](/2021/04/20/vpn-node-on-your-own-part-2.html) explains server side configuration.

## References
<b id="f1">1</b>. [Virtual Private Network](https://en.wikipedia.org/wiki/Virtual_private_network) [↩](#a1)<br/>
<b id="f2">2</b>. [Traefik v2](https://doc.traefik.io/traefik/) [↩](#a2)<br/>
<b id="f3">3</b>. [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) [↩](#a3)<br/>
<b id="f4">4</b>. [MTProto Mobile Protocol](https://core.telegram.org/mtproto) [↩](#a4)<br/>
<b id="f5">5</b>. [MTProto Fake TLS](https://geekbrit.org/content/22070) [↩](#a5)<br/>
<b id="f6">6</b>. [WireGuard: fast, modern, secure VPN tunnel](https://www.wireguard.com/) [↩](#a6)<br/>
<b id="f7">7</b>. [BIRD Internet Routing Daemon 2.0](https://bird.network.cz/) [↩](#a7)<br/>
<b id="f8">8</b>. [SD-WAN](https://en.wikipedia.org/wiki/SD-WAN) [↩](#a8)<br/>
<b id="f9">9</b>. [Customer Premises Equipment](https://en.wikipedia.org/wiki/Customer-premises_equipment) [↩](#a9)<br/>
<b id="f10">10</b>. [WireGuard does not focus on obfuscation](https://www.WireGuard.com/known-limitations/#deep-packet-inspection) [↩](#a10)<br/>
<b id="f11">11</b>. [Deep Packet Inspection](https://en.wikipedia.org/wiki/Deep_packet_inspection) [↩](#a11)<br/>
<b id="f12">12</b>. [QUIC transport layer network protocol](https://en.wikipedia.org/wiki/QUIC) [↩](#a12)<br/>
<b id="f13">13</b>. [Ubuntu Server 20.04 LTS](https://releases.ubuntu.com/20.04/) [↩](#a13)<br/>
<b id="f14">14</b>. [Docker](https://www.docker.com/) [↩](#a14)<br/>
<b id="f15">15</b>. [Docker Compose](https://docs.docker.com/compose/) [↩](#a15)<br/>
<b id="f16">16</b>. [Supervisor: a process control system](http://supervisord.org/) [↩](#a16)<br/>
<b id="f17">17</b>. [Docker Compose: support IPAM gateway in version 3.x](https://github.com/docker/compose/issues/6569) [↩](#a17)<br/>
<b id="f18">18</b>. [Podman](https://podman.io/) [↩](#a18)<br/>
<b id="f19">19</b>. [Traefik: Podman support](https://github.com/traefik/traefik/issues/5730) [↩](#a19)<br/>
<b id="f20">20</b>. [Border Gateway Protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) [↩](#a20)<br/>
<b id="f21">21</b>. [Bidirectional Forwarding Detection](https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection) [↩](#a21)<br/>
<b id="f23">22</b>. [OpenWrt](https://openwrt.org/) [↩](#a22)<br/>
