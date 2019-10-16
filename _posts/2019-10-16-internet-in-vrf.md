---
layout: post
title:  "Internet in VRF vs Internet in GRT"
tags: Internet BGP FullView VRF GRT
author: "riddler63"
---


## Internet in VRF vs Internet in GRT(inet.0) comparison

> **Disclaimer**: Isn't complete list of Pros and Cons.

Here is a comparison table for "Internet in VRF" and "Internet in GRT" options 

| Property/Issue      | Internet in VRF | Internet in GRT |
| ------------- | -------------   |-----------------|
| BGP Free Core                                 | Out of the box    | Have to use MPLS and /30 - /32 MPLS label allocation                |
| Prefix visibility at PE                       | Using unique RD per PE or BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) for VPNv4 and/or VPNv6 | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)                |
| Optimal path selection regardless of RR hierarchy  | Using unique RD per PE (increases RIB memory consumption)  | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) (increases RIB memory consumption) or  [BGP Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19) |
| [Route oscillation](https://tools.ietf.org/html/rfc3345) issue  | Using unique RD per PE  | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) together with [Advertise the Group Best Paths]( (https://tools.ietf.org/html/rfc7964#section-4))   |
| Flexible and simple approach to distinguish End-user, Transit, IXPs and Others | Built-in by RT import/export policy | Not Available |
| Flexible and simple approach to distribute routes partially | Built-in by RT import/export policy | BGP-ORF based on Prefix list or BGP Community (Nokia proprietary) policy |
| Traffic diversion for DDoS mitigation | Easy to diverse traffic by using RD and RT manipulation | Have to use BGP Flow-spec and other technics using protocols like BGP-LU |
| Customer isolation | Built-in | Not Available |
| Prefix filtration on Egress per BGP peer | Using RTC/RTF. Filtering can be done on RR |  BGP-ORF based on Prefix list policy on each PE|
| Low spec devices as Internet PE. Default route + partial BGP Full View (95% of traffic belongs to less than 500 prefixes) **(5)** | Using RTC/RTF and different RT   | BGP-ORF based on Prefix list on each PE |
| Ability to advertise Full feed, local prefix, IXP prefixes or any mix of above to the BGP peers | Simple and Flexible by using RT and BGP Communities | Complex BGP policies based on BGP communities |
| Strong demarcation between ISP infrastructure and Public Internet service | Built-in | N/A |
| IPv6 Internet | Using 6VPE (VPNv6 AFI/SAFI). Simple | Using 6PE (Complex BGP-LU configuration) and additional label in MPLS stack |
| BGP Fast Reroute | BGP PIC Core and BGP PIC Edge features | BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) and BGP FRR |
| RIB consumption | +8% + [~80% per Full View VRF ](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html) | No extra RIB memory consumption |
|FIB consumption | Some vendors might need (might not) extra X% FIB space to store VRF entries  | No extra FIB space needed |
| BGP security features | BGP RPKI, BGP FLow-spec and other BGP security features might not be available in VRF | Most of the BGP security features works in GRT |
| BGP convergence time | RTC/RTF will affect BGP convergence time | BGP ORF can affect BGP convergence time |
| Events to trigger BGP convergence | BGP events only (might be not fast enough). | BGP and IGP events (if NH address propagated via IGP)|
| BGP features | Vendors might have scarce feature set for BGP in VRF | Most of the BGP features will work in GRT |
| MPLS label allocation | Per Prefix - not enough MPLS labels; Per PE in VRF -> additional IP Lookup is needed on PE | Per Prefix - not enough MPLS labels; Per next-hop/interface |

Additional Links:

1. [Peering Fabric Design -> Peering and Internet in a VRF](https://xrdocs.io/design/blogs/latest-peering-fabric-hld#security)
2. [Is it safe to run Internet in a VRF?](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)
3. [Internet routing table in a vrf](https://seclists.org/nanog/2013/Mar/154)
4. [Service Provider Network Architecture - Internet in a VRF](https://www.reddit.com/r/networking/comments/5j8vg8/service_provider_network_architecture_internet_in/)
5. [Internet Traffic 2009 2019](https://www.youtube.com/watch?v=jGnVcCQUCdk&feature=youtu.be)
