---
layout: post
title:  "Internet in VRF vs Internet in GRT"
tags: Internet BGP FullView VRF GRT
author: "riddler63"
---

Internet in VRF vs Internet in GRT comparison

> **Disclaimer**: It's not a complete list of Pros and Cons.

Here is a comparison table for "Internet in VRF" and "Internet in GRT" options 

| Property/Issue      | Internet in VRF | Internet in GRT | Summary | 
|-----------------|-----------------|-----------------|-----------------| 
| BGP Free Core                                 | Out of the box    | Have to use MPLS and /30 - /32 MPLS label allocation                | VRF is better. Less action required for VRF | 
| Prefix visibility at PE                       | Using unique RD per PE or BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) for VPNv4 and/or VPNv6 | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911)                | Unique RD is out off the box functionality. No additional features required. | 
| Optimal path selection regardless of RR hierarchy  | Using unique RD per PE (increases RIB memory consumption) or [BGP Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19) for VPNv4/VPNv6  | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) (increases RIB memory consumption) or  [BGP Optimal Route Reflection](https://tools.ietf.org/html/draft-ietf-idr-bgp-optimal-route-reflection-19) | It depends. Unique RD might be better solution, until PE has scarce RIB resources | 
| [Route oscillation](https://tools.ietf.org/html/rfc3345) issue  | Using unique RD per PE  | Using BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) together with [Advertise the Group Best Paths]( (https://tools.ietf.org/html/rfc7964#section-4))   | VRF is better, no additional features required| 
| Flexible and simple approach to distinguish End-user, Transit, IXPs and Others | Built-in by RT import/export policy | Not Available | VRF is better | 
| Flexible and simple approach to distribute routes partially | Built-in by RT import/export policy | BGP-ORF based on Prefix list or BGP Community (Nokia only) policy | VRF is better. BGP-ORF introduces big configuration overhead | 
| Traffic diversion for DDoS mitigation | Easy to diverse traffic by using RD and RT manipulation. BGP Flowspec also an option | Have to use BGP Flowspec and other technics using protocols like BGP-LU | It depends. Traffic diversion by RT is less granular than BGP Flowspec. BGP Flowspec for VRF might not be supported by some vendors | 
| Customer isolation | Built-in | Not Available | VRF is better | 
| Prefix filtration on Egress per BGP peer | Using RTC/RTF. Filtering can be done on RR |  BGP-ORF based on Prefix list policy on each PE| RTC/RTF is better approach, but PE have to support certain BGP AFI/SAFI |
| Low spec devices as Internet PE. Default route + partial BGP Full View (95% of traffic belongs to less than 500 prefixes) **(5)** | Using RTC/RTF and different RT   | BGP-ORF based on Prefix list on each PE | VRF is better. BGP-ORF introduces big configuration overhead | 
| Ability to advertise Full View, local prefix, IXP prefixes or any mix of above to the BGP peers | Simple and flexible by using RT and BGP Communities | Complex BGP policies based on BGP communities | VRF is better, because it provides simple and flexible way to control prefix advertisment | 
| Strong demarcation between ISP infrastructure and Public Internet service | Built-in | N/A | VRF is better. | 
| IPv6 Internet | Using 6VPE (VPNv6 AFI/SAFI). Simple | Using 6PE (Complex BGP-LU configuration) and additional label in MPLS stack | VRF is better. It provides unified approach for both IPv4 and IPv6 | 
| BGP Fast Reroute | BGP PIC Core and BGP PIC Edge features | BGP [ADD-PATH](https://tools.ietf.org/html/rfc7911) and BGP FRR | It depends on features supported by the network | 
| RIB consumption | +8% + [~80% per Full View VRF ](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html) | No extra RIB memory consumption | GRT is better, no extra RIB resources consumption | 
|FIB consumption | Some vendors might need (might not) extra X% FIB space to store VRF entries  | No extra FIB space needed | GRT is better, no extra FIB resources consumption | 
| BGP security features | BGP RPKI, BGP FLow-spec and other BGP security features might not be available in VRF | Most of the BGP security features works in GRT | Currently GRT is better, BGP Security features for 99% implemented in GRT | 
| BGP convergence time | RTC/RTF will affect BGP convergence time | BGP ORF can affect BGP convergence time | It depends. In common case RTC/RTF will increase BGP convergence time | 
| Events to trigger BGP convergence | BGP events only (might not be fast enough). | BGP and IGP events (if NH address propagated via IGP)| It depends. BGP fast convergence might affect stability | 
| BGP features | Vendors might have scarce feature set for BGP in VRF | Most of the BGP features will work in GRT | Currently GRT is better.  | 
| MPLS label allocation | Per Prefix - not enough MPLS labels; Per NH/CE -> MPLS labels quantity depends on NH/CE quantity; Per PE in VRF -> additional IP Lookup is needed on PE | Per Prefix - not enough MPLS labels; Per NH/interface -> MPLS labels quantity depends on number of NH/interfaces | Platform depended. Different vendors uses different approach. Per NH/CE mpls label allocation works well for both options | 

Additional Links:

1. [Peering Fabric Design -> Peering and Internet in a VRF](https://xrdocs.io/design/blogs/latest-peering-fabric-hld#security)
2. [Is it safe to run Internet in a VRF?](https://blog.ipspace.net/2012/07/is-it-safe-to-run-internet-in-vrf.html)
3. [Internet routing table in a vrf](https://seclists.org/nanog/2013/Mar/154)
4. [Service Provider Network Architecture - Internet in a VRF](https://www.reddit.com/r/networking/comments/5j8vg8/service_provider_network_architecture_internet_in/)
5. [Internet Traffic 2009 2019](https://www.youtube.com/watch?v=jGnVcCQUCdk&feature=youtu.be)
