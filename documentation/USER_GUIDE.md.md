\# EVPNMultihomingForge Pro \- User Guide & CI/CD Pipeline Manual

\#\# 1\. Web Studio Quickstart

1\. Open \`source-code/index.html\` in any modern web browser.  
2\. Select an industry preset from the top dropdown:  
   \- \*\*Datacenter ToR LACP Arista (Grade A+)\*\*: 100G/400G dual-leaf Top-of-Rack deployment using Type 1 IEEE 802.1AX LACP ESI, RFC 8584 Preference-based DF election, Local Bias split-horizon, core-isolation tracking, and distributed ARP suppression.  
   \- \*\*Cloud Spine-Leaf MAC Junos (Grade A+)\*\*: Cloud spine-leaf multi-tenant fabric using Type 3 MAC-based ESI with MPLS ESI split-horizon label and cryptographic BGP peering authentication.  
   \- \*\*Hazardous Loop-Prone Split-Brain (Grade F)\*\*: Demonstrates failure modes of manual arbitrary ESI (Type 0), disabled split-horizon (broadcast loops), independent LACP System IDs (disabled LAG), and unauthenticated peering.  
3\. Review the \*\*10-Point Zero-Trust Scorecard\*\* to inspect any split-horizon loop, blackholing, or election flap risks.  
4\. Modify parameters (ESI Type, LACP MAC, Port Key, DF Preference, VNI Mapping).  
5\. Inspect the generated \*\*RFC 7432 10-Byte ESI Hex String\*\* and \*\*EVPN Route Type 1 (Ethernet Auto-Discovery) NLRI Hex Wireframe\*\*.  
6\. Switch synthesizer tabs to export production configuration:  
   \- \*\*Arista EOS\*\*: \`interface Port-Channel10\`, \`evpn ethernet-segment\`, \`identifier ...\`  
   \- \*\*Cisco Nexus NX-OS\*\*: \`interface port-channel10\`, \`ethernet-segment 1\`, \`esi ...\`  
   \- \*\*Juniper Junos\*\*: \`set interfaces ae10 esi ...\`  
   \- \*\*FRRouting (FRR) / Cumulus\*\*: \`interface bond0\`, \`evpn mh es-id ...\`  
   \- \*\*Verification Script\*\*: Bash/vtysh script checking next-hops, DF status, and local-bias filters.

\---

\#\# 2\. DevSecOps CLI Tool (\`evpnmultihomingforge-cli.py\`)

The standalone Python CLI tool requires zero external dependencies and runs on Python 3.8+.

\#\#\# 2.1 Encoding 10-Byte ESI  
\`\`\`bash  
python3 build/evpnmultihomingforge-cli.py encode-esi \-p presets/datacenter-tor-lacp-arista.json  
\`\`\`  
Output:  
\`\`\`  
\[RFC 7432 ESI ENCODE\] 10-Byte Value: 01:00:1c:73:00:00:01:00:64:00  
Type: 1 | Byte 0: 0x01  
\`\`\`

\#\#\# 2.2 Encoding EVPN Route Type 1 NLRI to Binary Hex  
\`\`\`bash  
python3 build/evpnmultihomingforge-cli.py encode-nlri \-p presets/datacenter-tor-lacp-arista.json  
\`\`\`

\#\#\# 2.3 Running CI/CD Pipeline Audits  
\`\`\`bash  
\# Clean production configuration \-\> Exits code 0  
python3 build/evpnmultihomingforge-cli.py audit \-p presets/datacenter-tor-lacp-arista.json \--strict

\# Insecure/fragile configuration \-\> Exits code 1 (Pipeline Fails)  
python3 build/evpnmultihomingforge-cli.py audit \-p presets/vulnerable-loop-prone-split-brain.json \--strict  
\`\`\`

\#\#\# 2.4 Synthesizing Router Configurations  
\`\`\`bash  
python3 build/evpnmultihomingforge-cli.py synthesize \-p presets/datacenter-tor-lacp-arista.json \-f arista  
python3 build/evpnmultihomingforge-cli.py synthesize \-p presets/cloud-spine-leaf-mac-junos.json \-f junos  
\`\`\`

\---

\#\# 3\. GitHub Actions CI/CD Pipeline Gate

Add this workflow to your NetDevOps repository (\`.github/workflows/evpn-multihoming-gate.yml\`):

\`\`\`yaml  
name: Zero-Trust EVPN Multihoming CI/CD Gate

on:  
  pull\_request:  
    paths:  
      \- 'evpn/\*\*'  
      \- 'fabric/\*\*'  
      \- 'routing/\*\*'

jobs:  
  evpn-audit:  
    runs-on: ubuntu-latest  
    steps:  
      \- name: Checkout Code  
        uses: actions/checkout@v4

      \- name: Set up Python  
        uses: actions/setup-python@v5  
        with:  
          python-version: '3.11'

      \- name: Audit EVPN Multihoming Configuration (Strict Gate)  
        run: |  
          python3 build/evpnmultihomingforge-cli.py audit \-p presets/datacenter-tor-lacp-arista.json \--strict  
\`\`\`  
