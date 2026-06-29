---
layout: post
title:  "Internet in VRF vs Internet in GRT"
tags: internet bgp vrf grt
author: "community"
---

A comparison of running the Internet in a VRF versus the Global Routing Table (GRT).

The comparison is split into two tables. The first covers **core distinctions** rooted in protocol behavior and network design — these are largely inherent to each approach, though new RFCs and mechanisms (for example, a mass-withdrawal mechanism for L3VPN) could shift some of them over time. The second covers **current vendor implementation status** — feature support that is vendor- and platform-dependent and may improve over time.

### Core distinctions (protocol and network design)

| Property/Issue      | Internet in VRF | Internet in GRT | Summary | 
|-----------------|-----------------|-----------------|-----------------| 
| Service BGP-free core | Out of the box | Requires MPLS and /30–/32 MPLS label allocation | VRF is better. Less effort required. | 
| Prefix visibility at the PE | Unique RD per PE, or BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1] for VPNv4 and/or VPNv6 (rarely supported by vendors) | BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1], or BGP RR [Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19)[^2] or BGP Full-Mesh/Partial Mesh | VRF is slightly better. Unique RD works out of the box; GRT needs ADD-PATH or ORR. | 
| Optimal path selection regardless of RR hierarchy | Unique RD per PE, or BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1] for VPNv4 and/or VPNv6 (rarely supported by vendors) | BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1] , or BGP RR [BGP Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19)[^2] or BGP Full-Mesh/Partial Mesh | VRF is slightly better. Unique RD works out of the box (higher RIB cost); GRT needs ADD-PATH or ORR. ORR works on both and avoids the RIB cost. | 
| [Route oscillation](https://tools.ietf.org/html/rfc3345)[^3] issue | Unique RD per PE | BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1] together with [Advertise the Group Best Paths](https://tools.ietf.org/html/rfc7964#section-4)[^4] | VRF is better. No additional features required. | 
| Flexible, simple way to distinguish End-users, Transit, IXPs and Others | Built in via RT import/export policy | Not available | VRF is better. | 
| Selective / partial prefix advertisement (full view / local / IXP) | Built in via RT import/export policy and BGP communities | BGP-ORF (prefix-list or BGP community, Nokia only), or complex BGP community policies | VRF is better. RT and communities give a simple, flexible way to advertise routes partially; GRT needs BGP-ORF (significant configuration overhead) or complex community policies. | 
| Traffic diversion for DDoS mitigation | Easy to divert traffic via RD and RT manipulation. BGP Flowspec is also an option. | Requires BGP Flowspec and other techniques using protocols such as BGP-LU | It depends. Traffic diversion by RT is less granular than BGP Flowspec. | 
| Customer isolation | Built in | Possible via ACLs or complex DCU/SCU policies | VRF is better. GRT requires ACLs or complex DCU/SCU policies to achieve the same isolation. | 
| Egress prefix filtering per BGP peer | RTC/RTF. Filtering can be performed on the RR. | BGP-ORF based on prefix-list policy on each PE | RTC/RTF is the better approach, but the PE must support the relevant BGP AFI/SAFI. | 
| Low-spec PE: default route + partial full view[^5] | RTC/RTF and different RTs | BGP-ORF based on prefix-list on each PE | VRF is better. BGP-ORF introduces significant configuration overhead. | 
| Strong demarcation between ISP infrastructure and public Internet service | Clear separation — infrastructure stays in GRT, Internet service lives in its own VRF | Usually mixes infrastructure and Internet service routes in the same table | VRF is better. Clean, built-in separation of infrastructure from service. | 
| Building virtual topologies | Built in — RT import/export shapes per-service topologies without touching the physical/BGP topology | Requires adjusting the BGP topology or applying complex filtering | VRF is better. RT makes virtual topologies simple and decoupled from the underlay. | 
| IPv6 Internet | 6VPE (VPNv6 AFI/SAFI). Simple. | 6PE (complex BGP-LU configuration) and an additional label in the MPLS stack | VRF is better. It provides a unified approach for both IPv4 and IPv6. | 
| Network fast convergence (data plane) | BGP PIC Edge works only if the entire ASBR goes down. If only the link to the peer fails, there is no mass-withdrawal trigger, so the ASBR may form a forwarding loop until BGP fully converges. | BGP-LU can create /32 routes to the eBGP next hop, providing a trigger to invalidate all routes from that peer (combined with BGP PIC Edge / [ADD-PATH](https://tools.ietf.org/html/rfc7911)[^1] and BGP FRR). Fast reroute works for both link and node failure. | GRT is better. In VRF, fast convergence is limited to full-ASBR failure; partial (link) failures risk transient forwarding loops until BGP converges. | 
| Next-hop resolution | More controlled NH resolution via auxiliary tables / tunnel-RIBs | Resolution may be confused by an IP routing table populated by multiple protocols; no unified way to control which routes are eligible for eBGP NH resolution | VRF is better. It provides a cleaner, more deterministic NH resolution process. | 
| BGP convergence (time and triggering events) | Driven by BGP events only (may not be fast enough); RTC/RTF affects convergence time | Driven by BGP and IGP events (if the next-hop is propagated via IGP); since GRT is a single shared table, BGP-LU can also trigger convergence for IPv4/IPv6 unicast. BGP-ORF can affect convergence time. | GRT is slightly better. It triggers convergence on port-down / eBGP-down; VRF relies on BGP events only. | 

### Current vendor implementation status (may change over time)

| Property/Issue      | Internet in VRF | Internet in GRT | Summary | 
|-----------------|-----------------|-----------------|-----------------| 
| RIB consumption | +8%, plus [~80% per full-view VRF](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)[^6]; actual overhead is vendor-specific | No extra RIB memory consumption | GRT is better. Extra RIB consumption in VRF is vendor-dependent. | 
| FIB consumption | Some vendors may (or may not) need extra FIB space to store VRF entries | No extra FIB space needed | GRT is better. No extra FIB resource consumption. | 
| BGP security features | BGP RPKI, BGP Flowspec and other BGP security features may not be available in VRF | Most BGP security features work in GRT | Currently GRT is better. BGP security features are ~99% implemented in GRT. | 
| Security feature maturity | Some security features may not work in VRF, as most vendors focus their implementation on Internet in GRT | Security features are typically implemented and validated in GRT first | GRT is better. Vendor focus on GRT means more mature, better-tested security features. | 
| Monitoring and telemetry (BMP, streaming telemetry) | BMP streaming and streaming telemetry may have limited support for VRF | Full support for BMP and streaming telemetry | GRT is better. VRF monitoring/telemetry support may be limited or vendor-dependent. | 
| Looking Glass tooling | Open-source Looking Glasses use GRT by default; VRF support may require customization | Works out of the box | GRT is better. VRF needs extra tooling customization. | 
| Traffic diversion (BGP Flowspec for VRF) | BGP Flowspec for VRF may not be supported by some vendors | Supported | GRT is better today, where Flowspec for VRF is unavailable. | 
| VPNv4 over IPv6 next hop ([RFC 8950](https://datatracker.ietf.org/doc/html/rfc8950)[^7]) | Support is patchy across vendors, so an IPv6-only core under the VRF can be hard to pull off | Same idea for plain IPv4, and far more widely supported | GRT is the safer bet today. | 
| BGP feature set | Vendors may offer a limited BGP feature set in VRF | Most BGP features work in GRT | Currently GRT is better. | 
| MPLS label allocation | Per prefix — not enough MPLS labels; per NH/CE — label count depends on NH/CE count; per PE in VRF — an additional IP lookup is needed on the PE | Per prefix — not enough MPLS labels; per NH/interface — label count depends on the number of NHs/interfaces | Platform-dependent. Different vendors use different approaches. Per-NH/CE MPLS label allocation works well for both options. | 

> **Note:** BGP ADD-PATH increases RIB consumption for both options (VRF and GRT), since each PE holds multiple paths per prefix instead of one.

## Glossary

| Term | Meaning |
|------|---------|
| **VRF** | Virtual Routing and Forwarding — an independent routing/forwarding instance on a router |
| **GRT** | Global Routing Table — the router's default (non-VRF) routing table |
| **BGP** | Border Gateway Protocol — the routing protocol used between autonomous systems |
| **PE** | Provider Edge — the SP router that connects to customers/peers |
| **CE** | Customer Edge — the customer router facing the PE |
| **RR** | Route Reflector — a BGP router that re-advertises routes to avoid full iBGP mesh |
| **RD** | Route Distinguisher — value prepended to a prefix to make VPN routes unique |
| **RT** | Route Target — BGP extended community controlling VPN route import/export |
| **RTC / RTF** | Route Target Constraint / Filtering — limits VPN routes sent to a PE to only those it needs |
| **RIB** | Routing Information Base — the control-plane routing table |
| **FIB** | Forwarding Information Base — the data-plane (hardware) forwarding table |
| **MPLS** | Multiprotocol Label Switching — label-based forwarding in the core |
| **BGP-LU** | BGP Labeled Unicast — distributes MPLS labels with IPv4/IPv6 prefixes via BGP |
| **NH** | Next Hop — the next router toward a destination |
| **ASBR** | Autonomous System Boundary Router — the router peering with another AS (eBGP edge) |
| **ADD-PATH** | BGP extension allowing advertisement of multiple paths for the same prefix ([RFC 7911](https://tools.ietf.org/html/rfc7911)) |
| **ORR** | BGP Optimal Route Reflection — lets an RR pick the best path from a client's perspective |
| **BGP-ORF** | BGP Outbound Route Filtering — a peer signals which prefixes it wants to receive |
| **Flowspec** | BGP Flow Specification — distributes traffic-filtering/diversion rules via BGP |
| **RPKI** | Resource Public Key Infrastructure — cryptographic validation of route origin |
| **BMP** | BGP Monitoring Protocol — streams BGP RIB state and updates to a monitoring station |
| **PIC** | Prefix Independent Convergence — fast reroute scheme (PIC Core / PIC Edge) |
| **FRR** | Fast Reroute — pre-computed backup paths for sub-second failover |
| **AFI / SAFI** | Address Family Identifier / Subsequent AFI — defines the address type a BGP session carries |
| **VPNv4 / VPNv6** | BGP address families carrying IPv4/IPv6 VPN (RD-qualified) routes |
| **6PE / 6VPE** | IPv6 over MPLS in the GRT (6PE) / in a VRF (6VPE) |
| **IXP** | Internet Exchange Point — shared peering fabric between networks |
| **IGP** | Interior Gateway Protocol — intra-AS routing protocol (e.g. OSPF, IS-IS) |
| **DDoS** | Distributed Denial of Service — attack flooding a target with traffic |
| **ACL** | Access Control List — packet filter matching on header fields |
| **DCU / SCU** | Destination / Source Class Usage — Juniper feature classifying traffic by destination/source prefix for filtering and accounting |
| **NLRI** | Network Layer Reachability Information — the prefix(es) carried in a BGP update |
| **Full view** | The complete Internet BGP routing table |

## Further reading

- [Peering Fabric Design → Peering and Internet in a VRF](https://xrdocs.io/design/blogs/latest-peering-fabric-hld#security)
- [Internet routing table in a VRF](https://seclists.org/nanog/2013/Mar/154)
- [Service Provider Network Architecture - Internet in a VRF](https://www.reddit.com/r/networking/comments/5j8vg8/service_provider_network_architecture_internet_in/)

## References

[^1]: [Advertisement of Multiple Paths in BGP](https://tools.ietf.org/html/rfc7911)
[^2]: [BGP Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19)
[^3]: [Border Gateway Protocol Persistent Route Oscillation Condition](https://tools.ietf.org/html/rfc3345)
[^4]: [Advertise the Group Best Paths](https://tools.ietf.org/html/rfc7964#section-4)
[^5]: [Internet Traffic 2009 2019](https://www.youtube.com/watch?v=jGnVcCQUCdk)
[^6]: [Is it safe to run Internet in a VRF?](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)
[^7]: [RFC 8950 - Advertising IPv4 NLRI with an IPv6 Next Hop](https://datatracker.ietf.org/doc/html/rfc8950)
