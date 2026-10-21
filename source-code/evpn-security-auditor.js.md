/\*\*  
 \* EVPNMultihomingForge Pro \- 10-Point Zero-Trust EVPN Multihoming & Loop-Free Fabric Health Auditor  
 \* Audits all-active Ethernet Segment configurations against split-horizon loop risks,  
 \* DF election desynchronization, core isolation, and convergence failures.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.EVPNSecurityAuditor \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  function auditConfiguration(config) {  
    const findings \= \[\];  
    let score \= 0;

    // 1\. ESI Formatting & Deterministic Type Validation  
    const esiType \= parseInt(config.esiType \!== undefined ? config.esiType : 1, 10);  
    if (esiType \=== 1 || esiType \=== 3\) {  
      score \+= 10;  
      findings.push({  
        id: 'ESI\_STANDARD\_TYPE',  
        title: \`Standard RFC 7432 ESI Type (\${esiType \=== 1 ? 'Type 1: LACP' : 'Type 3: MAC-based'})\`,  
        status: 'PASS',  
        points: 10,  
        desc: 'Deterministic ESI generation ensures seamless inter-chassis LACP / port-channel bonding across multi-vendor leaf switches.'  
      });  
    } else if (esiType \=== 0\) {  
      score \+= 6;  
      findings.push({  
        id: 'ESI\_STANDARD\_TYPE',  
        title: 'Manual Arbitrary ESI (Type 0)',  
        status: 'WARNING',  
        points: 6,  
        desc: 'Manual Type 0 ESI configured. Requires manual synchronization across leaves, risking typo-induced split-brain ESI mismatches.'  
      });  
    } else {  
      score \+= 8;  
      findings.push({  
        id: 'ESI\_STANDARD\_TYPE',  
        title: \`Alternative ESI (Type \${esiType})\`,  
        status: 'PASS',  
        points: 8,  
        desc: \`RFC 7432 ESI Type \${esiType} generated.\`  
      });  
    }

    // 2\. Split-Horizon Loop Prevention Architecture  
    if (config.splitHorizonType \=== 'LOCAL\_BIAS' || config.splitHorizonType \=== 'MPLS\_LABEL') {  
      score \+= 10;  
      findings.push({  
        id: 'SPLIT\_HORIZON\_PROTECTION',  
        title: \`Active Split-Horizon Filtering (\${config.splitHorizonType})\`,  
        status: 'PASS',  
        points: 10,  
        desc: \`Hardware split-horizon filtering active (\${config.splitHorizonType}). Completely prevents BUM traffic received from fabric from looping back to CE.\`  
      });  
    } else {  
      findings.push({  
        id: 'SPLIT\_HORIZON\_PROTECTION',  
        title: 'Missing or Disabled Split-Horizon Filter',  
        status: 'FAIL',  
        points: 0,  
        desc: 'CRITICAL: No split-horizon mechanism active. High risk of catastrophic BUM broadcast loops across all-active multihomed links.'  
      });  
    }

    // 3\. Designated Forwarder (DF) Election Algorithm  
    if (config.dfAlgorithm \=== 'PREFERENCE') {  
      score \+= 10;  
      findings.push({  
        id: 'DF\_ELECTION\_ALGORITHM',  
        title: 'RFC 8584 Preference-Based DF Election with Don\\'t Preempt (DP)',  
        status: 'PASS',  
        points: 10,  
        desc: 'Deterministic preference election with hold-off timers and DP flag prevents BUM broadcast duplicate bursts and election flaps.'  
      });  
    } else if (config.dfAlgorithm \=== 'MODULO') {  
      score \+= 6;  
      findings.push({  
        id: 'DF\_ELECTION\_ALGORITHM',  
        title: 'Legacy Modulo-Based DF Election (RFC 7432)',  
        status: 'WARNING',  
        points: 6,  
        desc: 'Modulo election active. When leaf reboots or peer joins, modulo re-mapping causes temporary BUM traffic drops across multiple VLANs.'  
      });  
    } else {  
      findings.push({  
        id: 'DF\_ELECTION\_ALGORITHM',  
        title: 'Unsynchronized DF Election',  
        status: 'FAIL',  
        points: 0,  
        desc: 'CRITICAL: No designated forwarder election configured. May result in either 0 forwarders (blackholing) or duplicate forwarders (broadcast storm).'  
      });  
    }

    // 4\. EVPN Route Type 1 Fast Convergence (Mass Withdraw & Aliasing)  
    if (config.enableType1Aliasing && config.enableType1MassWithdraw) {  
      score \+= 10;  
      findings.push({  
        id: 'TYPE1\_CONVERGENCE',  
        title: 'Full EVPN Route Type 1 Support (Aliasing & Mass Withdraw)',  
        status: 'PASS',  
        points: 10,  
        desc: 'Both per-EVI (aliasing load balancing) and per-ES (sub-second link-down mass withdraw) routes advertised to remote VTEPs.'  
      });  
    } else if (config.enableType1Aliasing || config.enableType1MassWithdraw) {  
      score \+= 6;  
      findings.push({  
        id: 'TYPE1\_CONVERGENCE',  
        title: 'Partial EVPN Route Type 1 Capability',  
        status: 'WARNING',  
        points: 6,  
        desc: 'Missing either aliasing or mass withdraw. Failover convergence will fall back to slow MAC-by-MAC Type 2 withdrawal.'  
      });  
    } else {  
      findings.push({  
        id: 'TYPE1\_CONVERGENCE',  
        title: 'Route Type 1 Ethernet A-D Disabled',  
        status: 'FAIL',  
        points: 0,  
        desc: 'Fast failover disabled. Loss of local access link will cause traffic blackholing until BGP withdraws all individual MACs.'  
      });  
    }

    // 5\. Core Isolation Link Protection  
    if (config.coreIsolationGuard) {  
      score \+= 10;  
      findings.push({  
        id: 'CORE\_ISOLATION\_GUARD',  
        title: 'Uplink Core-Isolation Tracking & Access Port Shutdown',  
        status: 'PASS',  
        points: 10,  
        desc: 'Leaf automatically suspends multihomed access links if all BGP EVPN spine uplinks fail, redirecting server traffic to healthy redundant leaf.'  
      });  
    } else {  
      findings.push({  
        id: 'CORE\_ISOLATION\_GUARD',  
        title: 'Core Isolation Protection Inactive',  
        status: 'FAIL',  
        points: 0,  
        desc: 'CRITICAL: If leaf loses spine connectivity, it continues attracting CE traffic without an egress path to the fabric, blackholing 50% of traffic.'  
      });  
    }

    // 6\. LACP System-ID & Port-Channel Synchronization  
    if (config.lacpSharedSystemId) {  
      score \+= 10;  
      findings.push({  
        id: 'LACP\_SYNCHRONIZATION',  
        title: 'Shared Virtual LACP System MAC & Priority',  
        status: 'PASS',  
        points: 10,  
        desc: 'Both leaf switches present identical LACP System ID to CE, enabling standard IEEE 802.1AX LAG bundling without proprietary MLAG.'  
      });  
    } else {  
      findings.push({  
        id: 'LACP\_SYNCHRONIZATION',  
        title: 'Independent LACP System IDs',  
        status: 'FAIL',  
        points: 0,  
        desc: 'Leaves advertise different LACP System IDs. CE device will suspend one member port, disabling active-active dual homing.'  
      });  
    }

    // 7\. MAC Flap Damping & Mobility Throttling  
    if (config.macFlapDampening) {  
      score \+= 10;  
      findings.push({  
        id: 'MAC\_MOBILITY\_GUARD',  
        title: 'MAC Flap Damping & Mobility Rate-Limiting',  
        status: 'PASS',  
        points: 10,  
        desc: 'Suppresses runaway MAC mobility sequence number increments during misconfigured loopback or duplicate IP/MAC collisions.'  
      });  
    } else {  
      findings.push({  
        id: 'MAC\_MOBILITY\_GUARD',  
        title: 'Unrestricted MAC Mobility Sequencing',  
        status: 'FAIL',  
        points: 0,  
        desc: 'Control plane vulnerable to BGP UPDATE storms if rogue CE or loop causes high-frequency MAC address bouncing between leaves.'  
      });  
    }

    // 8\. BGP EVPN Session Peering Authentication  
    if (config.bgpAuthType \=== 'TCP-AO' || config.bgpAuthType \=== 'KEYCHAIN') {  
      score \+= 10;  
      findings.push({  
        id: 'BGP\_AUTH\_HARDENING',  
        title: 'RFC 5925 TCP-AO / Cryptographic Key-Chain Peering',  
        status: 'PASS',  
        points: 10,  
        desc: 'BGP EVPN session protected with cryptographic message authentication, preventing spoofed BGP RST or route injection.'  
      });  
    } else if (config.bgpAuthType \=== 'MD5') {  
      score \+= 6;  
      findings.push({  
        id: 'BGP\_AUTH\_HARDENING',  
        title: 'Legacy TCP MD5 Peering Authentication',  
        status: 'WARNING',  
        points: 6,  
        desc: 'Legacy MD5 configured. Recommend migrating to RFC 5925 TCP-AO for modern datacenter fabrics.'  
      });  
    } else {  
      findings.push({  
        id: 'BGP\_AUTH\_HARDENING',  
        title: 'Unauthenticated BGP EVPN Peering',  
        status: 'FAIL',  
        points: 0,  
        desc: 'CRITICAL: EVPN control plane peering is unauthenticated. Vulnerable to BGP session reset attacks.'  
      });  
    }

    // 9\. Consistent Symmetric IRB & VNI Mapping  
    if (config.l2Vni && config.l3Vni && config.l2Vni \!== config.l3Vni) {  
      score \+= 10;  
      findings.push({  
        id: 'VNI\_MAPPING\_CONSISTENCY',  
        title: 'Consistent Symmetric IRB Architecture (L2VNI & L3VNI)',  
        status: 'PASS',  
        points: 10,  
        desc: \`Clean separation of Bridging (L2VNI \${config.l2Vni}) and Routing (L3VNI \${config.l3Vni}) ensures seamless inter-subnet forwarding.\`  
      });  
    } else {  
      findings.push({  
        id: 'VNI\_MAPPING\_CONSISTENCY',  
        title: 'Misconfigured or Overlapping VNI Mapping',  
        status: 'FAIL',  
        points: 0,  
        desc: 'L2VNI and L3VNI are overlapping or undefined. Inter-subnet routing will fail across the overlay.'  
      });  
    }

    // 10\. ARP/ND Suppression & BUM Reduction  
    if (config.arpNdSuppression) {  
      score \+= 10;  
      findings.push({  
        id: 'ARP\_ND\_SUPPRESSION',  
        title: 'Distributed ARP / ND Suppression Active',  
        status: 'PASS',  
        points: 10,  
        desc: 'Local leaf proxies ARP/ND requests using EVPN Type 2 MAC/IP table, eliminating BUM broadcast floods over multihomed links.'  
      });  
    } else {  
      findings.push({  
        id: 'ARP\_ND\_SUPPRESSION',  
        title: 'ARP / ND Suppression Disabled',  
        status: 'FAIL',  
        points: 0,  
        desc: 'ARP/ND requests flooded across fabric, consuming valuable access link bandwidth on multihomed segments.'  
      });  
    }

    let grade \= 'F';  
    if (score \>= 90\) grade \= 'A+';  
    else if (score \>= 80\) grade \= 'A';  
    else if (score \>= 70\) grade \= 'B';  
    else if (score \>= 50\) grade \= 'C';

    return {  
      score,  
      grade,  
      findings,  
      timestamp: new Date().toISOString()  
    };  
  }

  return {  
    auditConfiguration  
  };  
});  
