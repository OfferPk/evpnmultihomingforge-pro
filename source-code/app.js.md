/\*\*  
 \* EVPNMultihomingForge Pro \- Application State Orchestrator  
 \*/

document.addEventListener('DOMContentLoaded', () \=\> {  
  const PRESETS \= {  
    'datacenter-tor-lacp-arista': {  
      leafHostname: 'leaf01.ashburn.dc1',  
      localAs: 65101,  
      vtepIp: '198.51.100.11',  
      portChannelId: 10,  
      esiType: 1,  
      lacpMac: '00:1c:73:00:00:01',  
      lacpPortKey: 100,  
      splitHorizonType: 'LOCAL\_BIAS',  
      dfAlgorithm: 'PREFERENCE',  
      dfPreference: 500,  
      enableType1Aliasing: true,  
      enableType1MassWithdraw: true,  
      coreIsolationGuard: true,  
      lacpSharedSystemId: true,  
      lacpSystemMac: '00:1c:73:00:00:01',  
      macFlapDampening: true,  
      bgpAuthType: 'TCP-AO',  
      l2Vni: 10100,  
      l3Vni: 10200,  
      l2VniVlan: 10,  
      accessVlans: '10,20,30',  
      arpNdSuppression: true  
    },  
    'cloud-spine-leaf-mac-junos': {  
      leafHostname: 'leaf02.chicago.dc2',  
      localAs: 65102,  
      vtepIp: '198.51.100.12',  
      portChannelId: 20,  
      esiType: 3,  
      systemMac: '00:05:86:71:00:02',  
      localDiscriminator: 10,  
      splitHorizonType: 'MPLS\_LABEL',  
      dfAlgorithm: 'PREFERENCE',  
      dfPreference: 200,  
      enableType1Aliasing: true,  
      enableType1MassWithdraw: true,  
      coreIsolationGuard: true,  
      lacpSharedSystemId: true,  
      lacpSystemMac: '00:05:86:71:00:02',  
      macFlapDampening: true,  
      bgpAuthType: 'KEYCHAIN',  
      l2Vni: 20100,  
      l3Vni: 20200,  
      l2VniVlan: 20,  
      accessVlans: '20,40,60',  
      arpNdSuppression: true  
    },  
    'vulnerable-loop-prone-split-brain': {  
      leafHostname: 'leaf-fragile01',  
      localAs: 65000,  
      vtepIp: '192.0.2.1',  
      portChannelId: 1,  
      esiType: 0,  
      arbitraryHex: '00:00:00:00:00:00:00:00:00',  
      splitHorizonType: 'NONE',  
      dfAlgorithm: 'MODULO',  
      dfPreference: 0,  
      enableType1Aliasing: false,  
      enableType1MassWithdraw: false,  
      coreIsolationGuard: false,  
      lacpSharedSystemId: false,  
      lacpSystemMac: '',  
      macFlapDampening: false,  
      bgpAuthType: 'NONE',  
      l2Vni: 10100,  
      l3Vni: 10100,  
      l2VniVlan: 10,  
      accessVlans: '10',  
      arpNdSuppression: false  
    }  
  };

  let activeFormat \= 'arista';

  function getFormData() {  
    const esiType \= parseInt(document.getElementById('esiType').value, 10);  
    return {  
      leafHostname: document.getElementById('leafHostname').value,  
      localAs: parseInt(document.getElementById('localAs') ? document.getElementById('localAs').value : 65101, 10\) || 65101,  
      vtepIp: document.getElementById('vtepIp').value,  
      portChannelId: parseInt(document.getElementById('portChannelId').value, 10\) || 10,  
      esiType,  
      lacpMac: document.getElementById('lacpMac').value,  
      lacpPortKey: parseInt(document.getElementById('lacpPortKey').value, 10\) || 100,  
      systemMac: '00:1c:73:aa:bb:cc',  
      localDiscriminator: 1,  
      splitHorizonType: document.getElementById('splitHorizonType').value,  
      dfAlgorithm: document.getElementById('dfAlgorithm').value,  
      dfPreference: parseInt(document.getElementById('dfPreference').value, 10\) || 500,  
      bgpAuthType: document.getElementById('bgpAuthType').value,  
      l2Vni: parseInt(document.getElementById('l2Vni').value, 10\) || 10100,  
      l3Vni: parseInt(document.getElementById('l3Vni').value, 10\) || 10200,  
      l2VniVlan: 10,  
      accessVlans: document.getElementById('accessVlans').value,  
      lacpSystemMac: document.getElementById('lacpSystemMac').value,  
      enableType1Aliasing: document.getElementById('enableType1Aliasing').checked,  
      enableType1MassWithdraw: document.getElementById('enableType1MassWithdraw').checked,  
      coreIsolationGuard: document.getElementById('coreIsolationGuard').checked,  
      lacpSharedSystemId: document.getElementById('lacpSharedSystemId').checked,  
      macFlapDampening: document.getElementById('macFlapDampening').checked,  
      arpNdSuppression: document.getElementById('arpNdSuppression').checked  
    };  
  }

  function applyDataToForm(data) {  
    if (\!data) return;  
    document.getElementById('leafHostname').value \= data.leafHostname || '';  
    document.getElementById('vtepIp').value \= data.vtepIp || '';  
    document.getElementById('portChannelId').value \= data.portChannelId || 10;  
    document.getElementById('esiType').value \= data.esiType \!== undefined ? data.esiType : 1;  
    document.getElementById('lacpMac').value \= data.lacpMac || '00:1c:73:00:00:01';  
    document.getElementById('lacpPortKey').value \= data.lacpPortKey || 100;  
    document.getElementById('splitHorizonType').value \= data.splitHorizonType || 'NONE';  
    document.getElementById('dfAlgorithm').value \= data.dfAlgorithm || 'MODULO';  
    document.getElementById('dfPreference').value \= data.dfPreference || 500;  
    document.getElementById('bgpAuthType').value \= data.bgpAuthType || 'NONE';  
    document.getElementById('l2Vni').value \= data.l2Vni || 10100;  
    document.getElementById('l3Vni').value \= data.l3Vni || 10200;  
    document.getElementById('accessVlans').value \= data.accessVlans || '10,20,30';  
    document.getElementById('lacpSystemMac').value \= data.lacpSystemMac || '';  
    document.getElementById('enableType1Aliasing').checked \= \!\!data.enableType1Aliasing;  
    document.getElementById('enableType1MassWithdraw').checked \= \!\!data.enableType1MassWithdraw;  
    document.getElementById('coreIsolationGuard').checked \= \!\!data.coreIsolationGuard;  
    document.getElementById('lacpSharedSystemId').checked \= \!\!data.lacpSharedSystemId;  
    document.getElementById('macFlapDampening').checked \= \!\!data.macFlapDampening;  
    document.getElementById('arpNdSuppression').checked \= \!\!data.arpNdSuppression;  
    renderAll();  
  }

  function renderAudit(data) {  
    const auditRes \= EVPNSecurityAuditor.auditConfiguration(data);  
    const scoreBadge \= document.getElementById('scoreBadge');  
    const headerScoreBadge \= document.getElementById('headerScoreBadge');  
    const scoreText \= document.getElementById('scoreText');  
    const scoreSummary \= document.getElementById('scoreSummary');  
    const scoreBanner \= document.getElementById('scoreBanner');  
    const auditList \= document.getElementById('auditList');

    scoreBadge.textContent \= auditRes.grade;  
    headerScoreBadge.textContent \= \`SCORE: \${auditRes.score}/100 (\${auditRes.grade})\`;  
    scoreText.textContent \= \`Zero-Trust Multihoming Score: \${auditRes.score} / 100\`;

    if (auditRes.grade \=== 'A+' || auditRes.grade \=== 'A') {  
      scoreBadge.style.color \= '\#10b981';  
      scoreBadge.style.borderColor \= '\#10b981';  
      scoreBadge.style.background \= 'rgba(16, 185, 129, 0.2)';  
      scoreBanner.style.background \= 'rgba(16, 185, 129, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(16, 185, 129, 0.3)';  
      scoreSummary.textContent \= 'RFC 7432 Standard ESI • Local Bias Split-Horizon • RFC 8584 Preference DF • Core Tracking Active';  
    } else if (auditRes.grade \=== 'B' || auditRes.grade \=== 'C') {  
      scoreBadge.style.color \= '\#f59e0b';  
      scoreBadge.style.borderColor \= '\#f59e0b';  
      scoreBadge.style.background \= 'rgba(245, 158, 11, 0.2)';  
      scoreBanner.style.background \= 'rgba(245, 158, 11, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(245, 158, 11, 0.3)';  
      scoreSummary.textContent \= 'Warning: Modulo DF election or missing mass withdraw may cause convergence delays and packet drop.';  
    } else {  
      scoreBadge.style.color \= '\#ef4444';  
      scoreBadge.style.borderColor \= '\#ef4444';  
      scoreBadge.style.background \= 'rgba(239, 68, 68, 0.2)';  
      scoreBanner.style.background \= 'rgba(239, 68, 68, 0.1)';  
      scoreBanner.style.borderColor \= 'rgba(239, 68, 68, 0.3)';  
      scoreSummary.textContent \= 'CRITICAL: Missing split-horizon loop protection, independent LACP IDs, and unauthenticated peering.';  
    }

    auditList.innerHTML \= '';  
    auditRes.findings.forEach(f \=\> {  
      const item \= document.createElement('div');  
      item.className \= \`audit-item \${f.status}\`;  
      item.innerHTML \= \`  
        \<div\>  
          \<strong\>\${f.title}\</strong\>  
          \<p style="font-size: 0.75rem; color: \#94a3b8; margin-top: 2px;"\>\${f.desc}\</p\>  
        \</div\>  
        \<span style="font-weight: 700; font-family: monospace;"\>\${f.status} (\${f.points} pts)\</span\>  
      \`;  
      auditList.appendChild(item);  
    });  
  }

  function renderHex(data) {  
    const esiBytes \= EVPNEngine.encodeESI(data);  
    const nlriBytes \= EVPNEngine.encodeType1RouteNLRI({  
      ...data,  
      rdIp: data.vtepIp,  
      rdNumber: 1,  
      ethernetTagId: 0,  
      vni: data.l2Vni  
    });

    const hexBox \= document.getElementById('hexBox');  
    const esiStr \= EVPNEngine.formatEsiColon(esiBytes);  
    const dump \= EVPNEngine.formatHexDump(nlriBytes);

    hexBox.textContent \= \`// 10-Byte ESI: \${esiStr}\\n// RFC 7432 EVPN Route Type 1 (Ethernet Auto-Discovery) NLRI (\${nlriBytes.length} Bytes):\\n\` \+ dump;  
  }

  function renderConfig(data) {  
    const esiBytes \= EVPNEngine.encodeESI(data);  
    const configOutput \= document.getElementById('configOutput');  
    let code \= '';

    switch (activeFormat) {  
      case 'arista':  
        code \= EVPNConfigSynthesizer.synthesizeArista(data, esiBytes);  
        break;  
      case 'cisco':  
        code \= EVPNConfigSynthesizer.synthesizeCisco(data, esiBytes);  
        break;  
      case 'junos':  
        code \= EVPNConfigSynthesizer.synthesizeJunos(data, esiBytes);  
        break;  
      case 'frr':  
        code \= EVPNConfigSynthesizer.synthesizeFRR(data, esiBytes);  
        break;  
      case 'verify':  
        code \= EVPNConfigSynthesizer.synthesizeVerificationScript(data, esiBytes);  
        break;  
      default:  
        code \= '\! Select a format';  
    }

    configOutput.textContent \= code;  
  }

  function renderAll() {  
    const data \= getFormData();  
    renderAudit(data);  
    renderHex(data);  
    renderConfig(data);  
  }

  // Event Listeners  
  document.getElementById('presetSelect').addEventListener('change', (e) \=\> {  
    const presetKey \= e.target.value;  
    if (PRESETS\[presetKey\]) {  
      applyDataToForm(PRESETS\[presetKey\]);  
    }  
  });

  document.getElementById('btnResetPreset').addEventListener('click', () \=\> {  
    const presetKey \= document.getElementById('presetSelect').value;  
    if (PRESETS\[presetKey\]) {  
      applyDataToForm(PRESETS\[presetKey\]);  
    }  
  });

  document.getElementById('btnRecalculate').addEventListener('click', renderAll);

  // Form Inputs live re-render  
  document.querySelectorAll('input, select').forEach(elem \=\> {  
    elem.addEventListener('input', renderAll);  
    elem.addEventListener('change', renderAll);  
  });

  // Synthesizer Tabs  
  document.querySelectorAll('.tab-btn').forEach(btn \=\> {  
    btn.addEventListener('click', () \=\> {  
      document.querySelectorAll('.tab-btn').forEach(b \=\> b.classList.remove('active'));  
      btn.classList.add('active');  
      activeFormat \= btn.getAttribute('data-format');  
      renderConfig(getFormData());  
    });  
  });

  // Copy Buttons  
  document.getElementById('btnCopyHex').addEventListener('click', () \=\> {  
    const hex \= document.getElementById('hexBox').textContent;  
    navigator.clipboard.writeText(hex).then(() \=\> {  
      alert('EVPN NLRI Hex wireframe copied to clipboard\!');  
    });  
  });

  document.getElementById('btnCopyConfig').addEventListener('click', () \=\> {  
    const cfg \= document.getElementById('configOutput').textContent;  
    navigator.clipboard.writeText(cfg).then(() \=\> {  
      alert('Synthesized multihoming configuration copied to clipboard\!');  
    });  
  });

  // Initial render  
  renderAll();  
});  
