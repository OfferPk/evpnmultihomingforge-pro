/\*\*  
 \* EVPNMultihomingForge Pro \- Multi-Vendor EVPN Multihoming Configuration Synthesizer  
 \* Generates production all-active Ethernet Segment configurations across  
 \* Arista EOS, Cisco Nexus NX-OS, Juniper Junos, FRRouting EVPN, and Linux Cumulus.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.EVPNConfigSynthesizer \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  function formatEsiColon(bytes) {  
    return bytes.map(b \=\> b.toString(16).padStart(2, '0')).join(':');  
  }

  function formatEsiDot(bytes) {  
    // Cisco dot format: xxxx.xxxx.xxxx.xxxx.xxxx  
    const hex \= bytes.map(b \=\> b.toString(16).padStart(2, '0')).join('');  
    return \`\${hex.substr(0,4)}.\${hex.substr(4,4)}.\${hex.substr(8,4)}.\${hex.substr(12,4)}.\${hex.substr(16,4)}\`;  
  }

  function synthesizeArista(cfg, esiBytes) {  
    const lines \= \[\];  
    const esiStr \= formatEsiColon(esiBytes);  
    const po \= cfg.portChannelId || 10;  
    const pref \= cfg.dfPreference || 500;

    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`\! Arista EOS RFC 7432 / RFC 8365 EVPN All-Active Multihoming Configuration\`);  
    lines.push(\`\! Leaf: \${cfg.leafHostname || 'leaf01.dc1'} | ESI: \${esiStr}\`);  
    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`interface Port-Channel\${po}\`);  
    lines.push(\`   description "All-Active Multihomed CE Link"\`);  
    lines.push(\`   switchport mode trunk\`);  
    lines.push(\`   switchport trunk allowed vlan \${cfg.accessVlans || '10,20,30'}\`);  
    lines.push(\`   evpn ethernet-segment\`);  
    lines.push(\`      identifier \${esiStr}\`);  
    lines.push(\`      redundancy all-active\`);  
    if (cfg.dfAlgorithm \=== 'PREFERENCE') {  
      lines.push(\`      designated-forwarder election preference \${pref} dont-preempt\`);  
    }  
    if (cfg.lacpSharedSystemId) {  
      lines.push(\`   lacp system-id \${cfg.lacpSystemMac || '00:1c:73:00:00:01'}\`);  
    }  
    lines.push(\`\!\`);  
    lines.push(\`router bgp \${cfg.localAs || 65000}\`);  
    lines.push(\`   vlan \${cfg.l2VniVlan || 10}\`);  
    lines.push(\`      rd \${cfg.vtepIp || '198.51.100.1'}:\${cfg.l2Vni || 10100}\`);  
    lines.push(\`      route-target both \${cfg.localAs || 65000}:\${cfg.l2Vni || 10100}\`);  
    lines.push(\`      redistribute learned\`);  
    lines.push(\`   \!\`);  
    lines.push(\`   address-family evpn\`);  
    lines.push(\`      neighbor EVPN-OVERLAY-SPINES activate\`);  
    lines.push(\`      neighbor EVPN-OVERLAY-SPINES next-hop-unchanged\`);  
    lines.push(\`\!\`);  
    if (cfg.arpNdSuppression) {  
      lines.push(\`interface Vxlan1\`);  
      lines.push(\`   vxlan vlan \${cfg.l2VniVlan || 10} flood vtep learned\`);  
      lines.push(\`   vxlan arp-suppression\`);  
    }

    return lines.join('\\n');  
  }

  function synthesizeCisco(cfg, esiBytes) {  
    const lines \= \[\];  
    const esiDot \= formatEsiDot(esiBytes);  
    const po \= cfg.portChannelId || 10;  
    const pref \= cfg.dfPreference || 500;

    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`\! Cisco Nexus NX-OS EVPN Multihoming Configuration (All-Active)\`);  
    lines.push(\`\! Device: \${cfg.leafHostname || 'leaf01.dc1'} | ESI: \${esiDot}\`);  
    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`feature nv overlay\`);  
    lines.push(\`feature vn-segment-vlan-based\`);  
    lines.push(\`nv overlay evpn\`);  
    lines.push(\`evpn multihoming\`);  
    if (cfg.coreIsolationGuard) {  
      lines.push(\`  core-tracking\`);  
    }  
    lines.push(\`\!\`);  
    lines.push(\`interface port-channel\${po}\`);  
    lines.push(\`  description "Dual-Homed EVPN ESI Access Port-Channel"\`);  
    lines.push(\`  switchport mode trunk\`);  
    lines.push(\`  switchport trunk allowed vlan \${cfg.accessVlans || '10,20,30'}\`);  
    lines.push(\`  ethernet-segment 1\`);  
    lines.push(\`    esi \${esiDot}\`);  
    if (cfg.dfAlgorithm \=== 'PREFERENCE') {  
      lines.push(\`    df-election preference \${pref}\`);  
    }  
    lines.push(\`\!\`);  
    if (cfg.lacpSharedSystemId) {  
      lines.push(\`  lacp system-mac \${cfg.lacpSystemMac || '00:1c:73:00:00:01'}\`);  
    }  
    lines.push(\`\!\`);  
    lines.push(\`evpn\`);  
    lines.push(\`  vni \${cfg.l2Vni || 10100} l2\`);  
    lines.push(\`    rd auto\`);  
    lines.push(\`    route-target import auto\`);  
    lines.push(\`    route-target export auto\`);

    return lines.join('\\n');  
  }

  function synthesizeJunos(cfg, esiBytes) {  
    const lines \= \[\];  
    const esiStr \= formatEsiColon(esiBytes);  
    const ae \= \`ae\${cfg.portChannelId || 10}\`;  
    const pref \= cfg.dfPreference || 500;

    lines.push(\`\# \==============================================================================\`);  
    lines.push(\`\# Juniper Junos RFC 7432 EVPN All-Active Multihoming Configuration\`);  
    lines.push(\`\# Node: \${cfg.leafHostname || 'leaf01.dc1'} | ESI: \${esiStr}\`);  
    lines.push(\`\# \==============================================================================\`);  
    lines.push(\`set interfaces \${ae} aggregated-ether-options lacp active\`);  
    if (cfg.lacpSharedSystemId) {  
      lines.push(\`set interfaces \${ae} aggregated-ether-options lacp system-id \${cfg.lacpSystemMac || '00:1c:73:00:00:01'}\`);  
    }  
    lines.push(\`set interfaces \${ae} esi \${esiStr}\`);  
    lines.push(\`set interfaces \${ae} esi all-active\`);  
    if (cfg.dfAlgorithm \=== 'PREFERENCE') {  
      lines.push(\`set interfaces \${ae} esi df-election-type preference \${pref}\`);  
    }  
    lines.push(\`set interfaces \${ae} unit 0 family ethernet-switching interface-mode trunk\`);  
    lines.push(\`set interfaces \${ae} unit 0 family ethernet-switching vlan members \[ \${cfg.accessVlans || '10 20 30'} \]\`);  
    lines.push(\`\`);  
    lines.push(\`\# EVPN Protocol Configuration\`);  
    lines.push(\`set protocols evpn encap vxlan\`);  
    lines.push(\`set protocols evpn multihoming\`);  
    if (cfg.coreIsolationGuard) {  
      lines.push(\`set protocols evpn multihoming core-isolation link-tracking\`);  
    }  
    lines.push(\`set protocols evpn extended-vni-list \[ \${cfg.l2Vni || 10100} \${cfg.l3Vni || 10200} \]\`);

    return lines.join('\\n');  
  }

  function synthesizeFRR(cfg, esiBytes) {  
    const lines \= \[\];  
    const esiStr \= formatEsiColon(esiBytes);  
    const pref \= cfg.dfPreference || 500;

    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`\! FRRouting (FRR) / Cumulus Linux EVPN Multihoming Configuration\`);  
    lines.push(\`\! \==============================================================================\`);  
    lines.push(\`interface bond0\`);  
    lines.push(\` evpn mh es-id \${esiStr}\`);  
    if (cfg.dfAlgorithm \=== 'PREFERENCE') {  
      lines.push(\` evpn mh es-df-pref \${pref}\`);  
    }  
    if (cfg.lacpSharedSystemId) {  
      lines.push(\` evpn mh es-sys-mac \${cfg.lacpSystemMac || '00:1c:73:00:00:01'}\`);  
    }  
    lines.push(\`\!\`);  
    lines.push(\`router bgp \${cfg.localAs || 65000}\`);  
    lines.push(\` address-family l2vpn evpn\`);  
    lines.push(\`  neighbor FABRIC activate\`);  
    lines.push(\`  advertise-all-vni\`);  
    lines.push(\`  autort both\`);  
    lines.push(\` exit-address-family\`);

    return lines.join('\\n');  
  }

  function synthesizeVerificationScript(cfg, esiBytes) {  
    const esiStr \= formatEsiColon(esiBytes);  
    return \`\#\!/bin/bash  
\# \==============================================================================  
\# EVPN Multihoming Health Verification Script  
\# Target ESI: \${esiStr}  
\# \==============================================================================

echo "\[\*\] Checking EVPN Ethernet Segment Status..."  
vtysh \-c "show evpn next-hops"  
vtysh \-c "show evpn es \${esiStr}"  
vtysh \-c "show evpn es-evi"

echo "\[\*\] Verifying Designated Forwarder (DF) Election..."  
vtysh \-c "show evpn es \${esiStr} detail" | grep \-E "(DF|Preference|Algorithm)"

echo "\[\*\] Verifying Split-Horizon Filter..."  
vtysh \-c "show evpn next-hops" | grep \-i "local-bias"  
echo "\[+\] Verification Complete."  
\`;  
  }

  return {  
    synthesizeArista,  
    synthesizeCisco,  
    synthesizeJunos,  
    synthesizeFRR,  
    synthesizeVerificationScript  
  };  
});  
