---
layout: post
title:  "Ethernet II frame"
tags: layer2
---

Ethernet II packet and frame structure

![Ethernet II frame](/images/ethernet-frame.png)

- **Preamble** - the preamble of an Ethernet packet consists of a 56-bit (seven-byte) pattern of alternating 1 and 0 bits, allowing devices on the network to easily synchronize their receiver clocks
- **SFD** - a 1 byte value that marks the end of the preamble, which is the first field of an Ethernet packet, and indicates the beginning of the Ethernet frame
- **DST MAC** - a receiver of a frame presented as MAC address
- **SRC MAC** - a sender of a frame presented as MAC address
- **Payload** - The minimum payload is 42 octets when an 802.1Q tag is present and 46 octets when absent. The maximum payload is 1500 octets. Non-standard jumbo frames allow for larger maximum payload size.
- **FCS** - allows detection of corrupted data within the entire frame as received on the receiver side. The FCS value is computed as a function of all fields except the FCS
- **Interpacket gap** - Interpacket gap is idle time between packets. After a packet has been sent, transmitters are required to transmit a minimum of 96 bits (12 octets) of idle line state before transmitting the next packet.
