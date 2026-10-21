/\*\*  
 \* EVPNMultihomingForge Pro \- RFC 7432 / RFC 8365 BGP EVPN Multihoming Engine  
 \* Generates 10-byte ESI Types 0-5, EVPN Type 1 Ethernet A-D NLRI byte streams,  
 \* RFC 8584 Designated Forwarder (DF) election tables, and Wireshark-style hex dumps.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.EVPNEngine \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  function macToBytes(macStr) {  
    const clean \= (macStr || '00:00:00:00:00:00').replace(/\[^0-9a-fA-F\]/g, '');  
    if (clean.length \!== 12\) return \[0, 0, 0, 0, 0, 0\];  
    const bytes \= \[\];  
    for (let i \= 0; i \< 12; i \+= 2\) {  
      bytes.push(parseInt(clean.substr(i, 2), 16));  
    }  
    return bytes;  
  }

  function ipToBytes(ipStr) {  
    const parts \= (ipStr || '0.0.0.0').trim().split('.');  
    if (parts.length \!== 4\) return \[0, 0, 0, 0\];  
    return parts.map(p \=\> {  
      const n \= parseInt(p, 10);  
      return isNaN(n) ? 0 : Math.min(255, Math.max(0, n));  
    });  
  }

  function uint16ToBytes(val) {  
    const n \= Math.max(0, Math.min(0xffff, parseInt(val, 10\) || 0));  
    return \[(n \>\> 8\) & 0xff, n & 0xff\];  
  }

  function uint24ToBytes(val) {  
    const n \= Math.max(0, Math.min(0xffffff, parseInt(val, 10\) || 0));  
    return \[(n \>\> 16\) & 0xff, (n \>\> 8\) & 0xff, n & 0xff\];  
  }

  function uint32ToBytes(val) {  
    const n \= Math.max(0, Math.min(0xffffffff, parseInt(val, 10\) || 0));  
    return \[  
      (n \>\>\> 24\) & 0xff,  
      (n \>\>\> 16\) & 0xff,  
      (n \>\>\> 8\) & 0xff,  
      n & 0xff  
    \];  
  }

  /\*\*  
   \* Generates a valid 10-byte (80-bit) RFC 7432 Ethernet Segment Identifier (ESI)  
   \*/  
  function encodeESI(options) {  
    const opts \= options || {};  
    const esiType \= parseInt(opts.esiType \!== undefined ? opts.esiType : 1, 10);  
    const bytes \= \[esiType & 0xff\];

    switch (esiType) {  
      case 0: { // Type 0: Arbitrary 9 bytes  
        const hex \= (opts.arbitraryHex || '00:11:22:33:44:55:66:77:88').replace(/\[^0-9a-fA-F\]/g, '');  
        for (let i \= 0; i \< 18; i \+= 2\) {  
          bytes.push(parseInt(hex.substr(i, 2\) || '00', 16));  
        }  
        break;  
      }  
      case 1: { // Type 1: CE LACP MAC (6B) \+ Port Key (2B) \+ Reserved (1B)  
        const mac \= macToBytes(opts.lacpMac || '00:1c:73:00:00:01');  
        const key \= uint16ToBytes(opts.lacpPortKey || 100);  
        bytes.push(...mac, ...key, 0x00);  
        break;  
      }  
      case 2: { // Type 2: Layer 2 Bridge MAC (6B) \+ Priority (2B) \+ Reserved (1B)  
        const mac \= macToBytes(opts.bridgeMac || '00:1c:73:00:00:02');  
        const prio \= uint16ToBytes(opts.bridgePriority || 32768);  
        bytes.push(...mac, ...prio, 0x00);  
        break;  
      }  
      case 3: { // Type 3: System MAC (6B) \+ Local Discriminator (3B)  
        const mac \= macToBytes(opts.systemMac || '00:1c:73:aa:bb:cc');  
        const disc \= uint24ToBytes(opts.localDiscriminator || 1);  
        bytes.push(...mac, ...disc);  
        break;  
      }  
      case 4: { // Type 4: Router ID (4B) \+ Local Discriminator (4B) \+ Reserved (1B)  
        const rId \= ipToBytes(opts.routerId || '198.51.100.1');  
        const disc \= uint32ToBytes(opts.localDiscriminator || 1);  
        bytes.push(...rId, ...disc, 0x00);  
        break;  
      }  
      case 5: { // Type 5: Autonomous System (4B) \+ Local Discriminator (4B) \+ Reserved (1B)  
        const asBytes \= uint32ToBytes(opts.asn || 65000);  
        const disc \= uint32ToBytes(opts.localDiscriminator || 1);  
        bytes.push(...asBytes, ...disc, 0x00);  
        break;  
      }  
      default: {  
        for (let i \= 0; i \< 9; i++) bytes.push(0x00);  
      }  
    }

    // Ensure exactly 10 bytes  
    while (bytes.length \< 10\) bytes.push(0x00);  
    return bytes.slice(0, 10);  
  }

  function formatEsiColon(esiBytes) {  
    return esiBytes.map(b \=\> b.toString(16).padStart(2, '0')).join(':');  
  }

  /\*\*  
   \* Encodes an EVPN Route Type 1: Ethernet Auto-Discovery (A-D) Route NLRI (RFC 7432 Section 7.1)  
   \*/  
  function encodeType1RouteNLRI(options) {  
    const opts \= options || {};  
    // Route Distinguisher (8 bytes) \- Type 1: IP(4B) \+ Assigned(2B) or AS(2B) \+ Assigned(4B)  
    const rdIp \= ipToBytes(opts.rdIp || '198.51.100.1');  
    const rdNum \= uint16ToBytes(opts.rdNumber || 1);  
    const rd \= \[0x00, 0x01, ...rdIp, ...rdNum\]; // Type 1 RD (8 bytes)

    // ESI (10 bytes)  
    const esi \= encodeESI(opts);

    // Ethernet Tag ID (4 bytes) \- 0 for per-ES route, or VLAN ID for per-EVI  
    const tagId \= uint32ToBytes(opts.ethernetTagId || 0);

    // MPLS Label / VNI (3 bytes) \- 20 bits label \+ 3 bits exp \+ 1 bit BoS  
    const vni \= parseInt(opts.vni || opts.mplsLabel || 10100, 10);  
    const labelBytes \= \[  
      (vni \>\> 12\) & 0xff,  
      (vni \>\> 4\) & 0xff,  
      ((vni & 0x0f) \<\< 4\) | 0x01 // BoS bit set  
    \];

    // EVPN NLRI Header:  
    // Route Type: 1 byte (0x01 \= Ethernet Auto-Discovery)  
    // Length: 1 byte (Length of route in bytes: 8 \+ 10 \+ 4 \+ 3 \= 25 bytes)  
    const nlriPayload \= \[...rd, ...esi, ...tagId, ...labelBytes\];  
    return \[0x01, nlriPayload.length, ...nlriPayload\];  
  }

  /\*\*  
   \* RFC 8584 Designated Forwarder (DF) Election Calculator  
   \*/  
  function calculateDFElection(peList, vlanList, algorithm) {  
    // peList: \[{ ip: '198.51.100.1', preference: 500, dp: true }, { ip: '198.51.100.2', preference: 100, dp: false }\]  
    const results \= {};  
    const sortedPes \= \[...peList\].sort((a, b) \=\> a.ip.localeCompare(b.ip));

    vlanList.forEach(vlan \=\> {  
      if (algorithm \=== 'PREFERENCE') {  
        // RFC 8584 Preference: Highest preference wins. Tie-break: highest IP.  
        const winner \= \[...peList\].sort((a, b) \=\> {  
          if (b.preference \!== a.preference) return b.preference \- a.preference;  
          return b.ip.localeCompare(a.ip);  
        })\[0\];  
        results\[vlan\] \= {  
          dfPe: winner.ip,  
          algorithm: 'RFC 8584 Preference-Based',  
          preference: winner.preference,  
          nonPreempt: \!\!winner.dp  
        };  
      } else {  
        // RFC 7432 Default Modulo: V mod N  
        const index \= vlan % sortedPes.length;  
        results\[vlan\] \= {  
          dfPe: sortedPes\[index\].ip,  
          algorithm: 'RFC 7432 Modulo-Based',  
          index  
        };  
      }  
    });

    return results;  
  }

  /\*\*  
   \* Formats byte array into Wireshark-style hex dump  
   \*/  
  function formatHexDump(byteArray) {  
    if (\!byteArray || \!byteArray.length) return '';  
    const lines \= \[\];  
    const bytesPerLine \= 16;

    for (let i \= 0; i \< byteArray.length; i \+= bytesPerLine) {  
      const slice \= byteArray.slice(i, i \+ bytesPerLine);  
      const offset \= i.toString(16).padStart(4, '0');  
        
      const hexPart \= slice  
        .map(b \=\> b.toString(16).padStart(2, '0'))  
        .join(' ');  
      const paddedHex \= hexPart.padEnd(bytesPerLine \* 3, ' ');

      const asciiPart \= slice  
        .map(b \=\> (b \>= 32 && b \<= 126 ? String.fromCharCode(b) : '.'))  
        .join('');

      lines.push(\`\${offset}:  \${paddedHex} |\${asciiPart}|\`);  
    }

    return lines.join('\\n');  
  }

  return {  
    encodeESI,  
    formatEsiColon,  
    encodeType1RouteNLRI,  
    calculateDFElection,  
    formatHexDump  
  };  
});  
