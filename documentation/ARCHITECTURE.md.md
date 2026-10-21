\# EVPNMultihomingForge Pro (v1.0.0) \- Technical Architecture Specification

\#\# 1\. System Overview

BGP EVPN all-active multi-homing (RFC 7432 and RFC 8365\) provides standard-based multi-chassis link aggregation (ESI-LAG) without proprietary inter-chassis peering links (such as Cisco vPC peer-link or Arista MLAG peer-link). 

Two or more ToR (Top-of-Rack) leaf switches present a single virtual Ethernet Segment Identifier (ESI) to a dual-homed server or access switch. Traffic sent by the CE across the LACP bundle is hashed across all active leaf links, providing true active-active load balancing, sub-second failover, and loop-free forwarding.

\*\*EVPNMultihomingForge Pro\*\* addresses the primary causes of EVPN multihoming outages:  
\- \*\*ESI Byte-Order Mismatches\*\*: Inconsistent ESI type generation across Leaf-1 and Leaf-2 leading to severed LACP aggregation.  
\- \*\*Split-Horizon BUM Loops\*\*: Forwarding BUM traffic received from the VXLAN/MPLS overlay back down to the CE over the redundant member link.  
\- \*\*Designated Forwarder (DF) Election Flaps\*\*: Preemption storms and duplicate forwarder states during spine link flaps.  
\- \*\*Silent Core-Isolation Blackholes\*\*: Failing to shut down downstream access links when all spine uplinks fail.

\---

\#\# 2\. RFC 7432 10-Byte ESI Formats

\`\`\`  
 0                   1                   2                   3  
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1  
\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
|   ESI Type    |                 ESI Content (9 Octets)        |  
\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
\`\`\`

\#\#\# Supported ESI Types:  
\- \*\*Type 0 (Arbitrary)\*\*: \`00:xx:xx:xx:xx:xx:xx:xx:xx:xx\` (Operator managed).  
\- \*\*Type 1 (IEEE 802.1AX LACP)\*\*: \`01:CE-MAC(6B):Port-Key(2B):00\`.  
\- \*\*Type 2 (Layer 2 Bridge)\*\*: \`02:Bridge-MAC(6B):Priority(2B):00\`.  
\- \*\*Type 3 (MAC-Based)\*\*: \`03:System-MAC(6B):Local-Discriminator(3B)\`.  
\- \*\*Type 4 (Router-ID Based)\*\*: \`04:Router-ID(4B):Local-Discriminator(4B):00\`.  
\- \*\*Type 5 (AS-Based)\*\*: \`05:ASN(4B):Local-Discriminator(4B):00\`.

\---

\#\# 3\. RFC 7432 Route Type 1: Ethernet Auto-Discovery NLRI

\`\`\`  
\+---------------------------------------+  
|  Route Distinguisher (8 Octets)       |  
\+---------------------------------------+  
|  Ethernet Segment Identifier (10 Oct) |  
\+---------------------------------------+  
|  Ethernet Tag ID (4 Octets)           |  
\+---------------------------------------+  
|  MPLS Label / VNI (3 Octets)          |  
\+---------------------------------------+  
\`\`\`  
\- \*\*Per-ES Route\*\*: Advertised with Tag ID \= 0\. Carries ESI Split-Horizon label (in MPLS/SRv6) and triggers fast Mass Withdraw upon access port failure.  
\- \*\*Per-EVI Route\*\*: Advertised with specific VLAN/EVI Tag ID. Enables multi-destination aliasing and load-balancing across multi-chassis leaves.

\---

\#\# 4\. 10-Point Zero-Trust Multihoming Rubric

| \# | Standard | Verification Metric | Weight |  
|---|----------|---------------------|--------|  
| 1 | \*\*RFC 7432 Standard ESI\*\* | Deterministic Type 1 (LACP) or Type 3 (MAC) 10-byte encoding | 10 pts |  
| 2 | \*\*Split-Horizon Loop Guard\*\* | Local Bias (RFC 8365\) or ESI Label (RFC 7432\) active | 10 pts |  
| 3 | \*\*RFC 8584 Preference DF\*\* | Preference election with Don't Preempt (DP) prevents duplicate BUM | 10 pts |  
| 4 | \*\*Route Type 1 Convergence\*\* | Both Aliasing (per-EVI) and Mass Withdraw (per-ES) active | 10 pts |  
| 5 | \*\*Core Isolation Link Tracking\*\* | Uplink tracking drops local access links if spines fail | 10 pts |  
| 6 | \*\*Shared LACP System ID\*\* | Virtual shared LACP MAC & Priority presented to CE | 10 pts |  
| 7 | \*\*MAC Flap Dampening\*\* | Mobility sequence throttling prevents control plane CPU exhaustion | 10 pts |  
| 8 | \*\*BGP Peering Auth\*\* | RFC 5925 TCP-AO or cryptographic key-chain peering | 10 pts |  
| 9 | \*\*Symmetric IRB Consistency\*\* | Distinct, synchronized L2VNI and L3VNI mappings | 10 pts |  
| 10 | \*\*Distributed ARP/ND Suppression\*\*| Local proxying eliminates BUM broadcast floods over access link | 10 pts |  
