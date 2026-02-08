# Cisco Viptela SD-WAN (CLI-Only, No vManage)

Cisco Viptela SD-WAN overlay implementation over MPLS underlay, built entirely via CLI without vManage. Demonstrates controller-based SD-WAN architecture with manual Enterprise Root CA certificate management.

---

## 🔬 Overview

This lab builds a Viptela SD-WAN fabric on top of an existing MPLS L3VPN underlay, using only CLI (no vManage GUI). It demonstrates the full controller-based SD-WAN lifecycle:

- **Controller Separation** – vBond (orchestrator), vSmart (control plane), vEdge (data plane)
- **Enterprise Root CA** – Manual certificate creation, signing, and installation via OpenSSL
- **Whitelist Registration** – Manual device authentication (normally automated by vManage)
- **OMP Route Exchange** – Overlay route distribution through vSmart controller
- **IPSec Data Plane** – Encrypted site-to-site tunnels with BFD monitoring

**【日本語サマリ】**<BR>
MPLS L3VPN上にViptela SD-WANオーバーレイをCLI onlyで構築。<BR>
証明書・ホワイトリスト・OMP・IPSec/BFDまでの全工程をvManageなしで実施。<BR>
制御プレーン分離アーキテクチャの動作を検証。

---

## 🏗️ Architecture

### Topology

<img width="500" alt="image" src="https://github.com/user-attachments/assets/af04d91a-240b-432c-8089-b2c4f0ef13d3" />

### Protocol Stack Comparison: FortiGate vs Viptela

| Function | FortiGate SD-WAN | Viptela SD-WAN |
|---|---|---|
| **Data Encryption** | IPSec | IPSec |
| **Route Exchange** | BGP / Static | OMP (via vSmart) |
| **Path Monitoring** | Health-check (ping/HTTP) | BFD (delay/jitter/loss) |
| **Control Plane** | Embedded (single appliance) | DTLS (vEdge ↔ vBond/vSmart) |
| **Management** | FortiManager (optional) | vManage (optional) |
| **Design Philosophy** | All-in-one appliance | Controller-separated (SDN) |

### Layered Architecture

| Layer | Control Plane (Route Info) | Data Plane (Packet Forwarding) |
|---|---|---|
| **Overlay (SD-WAN)** | OMP | IPSec tunnel |
| **Underlay (MPLS)** | BGP / OSPF / LDP | CEF + MPLS label switching |

**Data flow:**
```
Site1 → vEdge02 →[IPSec]→ CE1 →[CEF]→ PE1 →[MPLS]→ PE2 →[CEF]→ CE2 →[IPSec]→ vEdge10 → Site2
```

### Viptela Controller Roles

| Component | Role | Analogy |
|---|---|---|
| **vBond** | Authentication & orchestration (device whitelist, initial connection brokering) | Gatekeeper |
| **vSmart** | Control plane (OMP route distribution, policy enforcement) | Brain |
| **vManage** | Management plane (GUI, templates, monitoring) – *not used in this lab* | Dashboard |
| **vEdge** | Data plane (IPSec tunnels, packet forwarding) | Hands & feet |

**【日本語サマリ】**<BR>
FortiGateは1台完結型、Viptelaはコントローラ分離型（SDN）。<BR>
Overlay（OMP/IPSec）とUnderlay（BGP/MPLS）の2層構造でデータを転送。

---

## 📋 IP Addressing

### Underlay (MPLS)

| Link | Subnet | Device A | Device B |
|---|---|---|---|
| PE1–PE2 | 10.200.1.0/30 | PE1: .1 | PE2: .2 |
| PE1–CE1 | 10.101.1.0/30 | PE1: .1 | CE1: .2 |
| PE2–CE2 | 10.102.1.0/30 | PE2: .1 | CE2: .2 |

### Overlay (Viptela)

| Link | Subnet | Device A | Device B |
|---|---|---|---|
| CE1–vEdge02 | 10.1.1.0/30 | CE1 e0/1: .1 | vEdge02 ge0/0: .2 |
| CE1–vBond | 10.1.2.0/30 | CE1 e0/2: .1 | vBond ge0/0: .2 |
| CE1–vSmart | 10.1.3.0/30 | CE1 e0/3: .1 | vSmart eth0: .2 |
| CE2–vEdge10 | 10.200.2.0/24 | CE2 e0/1: .1 | vEdge10 ge0/0: .2 |

### Viptela System Parameters

| Device | System-IP | Site-ID | Org | Mgmt (VPN 512) |
|---|---|---|---|---|
| vBond | 10.10.10.1 | 1000 | Lab11 | 192.168.133.10 |
| vSmart | 10.10.10.2 | 1000 | Lab11 | 192.168.133.11 |
| vEdge02 | 10.10.10.3 | 1 | Lab11 | 192.168.133.12 |
| vEdge10 | 10.10.10.4 | 2 | Lab11 | 192.168.133.13 |

**【日本語サマリ】**<BR>
Underlay（MPLS）とOverlay（Viptela）のIPアドレス一覧。<BR>
VPN 0=Transport、VPN 1=Service、VPN 512=Management。

---

## 🔐 Enterprise Root CA (Manual, No vManage)

Without vManage, certificate management must be done entirely via CLI and OpenSSL. This is the process that vManage normally automates behind the scenes.

### Step 1: Create Root CA on EVE-NG host

Generate a 2048-bit RSA private key and a self-signed Root CA certificate valid for 10 years.

```bash
mkdir -p /root/CA && cd /root/CA
openssl genrsa -out CA.key 2048
openssl req -x509 -new -nodes -key CA.key -sha256 -days 3650 \
  -out CA.pem -subj "/C=JP/O=Lab11/CN=Lab11-Root-CA"
```

### Step 2: Distribute Root CA to all Viptela nodes

Transfer the CA certificate to each device via SCP using the management network (VPN 512 / Cloud0).

```bash
scp CA.pem admin@192.168.133.10:/home/admin/CA.pem   # vBond
scp CA.pem admin@192.168.133.11:/home/admin/CA.pem   # vSmart
scp CA.pem admin@192.168.133.12:/home/admin/CA.pem   # vEdge02
scp CA.pem admin@192.168.133.13:/home/admin/CA.pem   # vEdge10
```

### Step 3: Install Root CA on each device

Install the Root CA certificate chain so each device trusts certificates signed by this CA.

```
request root-cert-chain install /home/admin/CA.pem
→ "Successfully installed the root certificate chain"
```

### Step 4: Generate CSR on each device

Generate a Certificate Signing Request on each node. Enter "Lab11" as the organization-unit name when prompted.

```
request csr upload /home/admin/<device>.csr
```

### Step 5: Sign CSRs on EVE-NG host

Retrieve the CSRs via SCP, then sign each one with the Root CA to produce device certificates.

```bash
# Retrieve CSRs
scp admin@192.168.133.10:/home/admin/vBond.csr .
scp admin@192.168.133.11:/home/admin/vSmart.csr .
scp admin@192.168.133.12:/home/admin/vEdge02.csr .
scp admin@192.168.133.13:/home/admin/vEdge10.csr .

# Sign each CSR
openssl x509 -req -in vBond.csr -CA CA.pem -CAkey CA.key \
  -CAcreateserial -out vBond.crt -days 3650 -sha256
# (repeat for vSmart.csr, vEdge02.csr, vEdge10.csr)
```

### Step 6: Install signed certificates on each device

Transfer the signed certificates back to each device and install them.

```bash
scp vBond.crt admin@192.168.133.10:/home/admin/vBond.crt
# (repeat for vSmart, vEdge02, vEdge10)
```

```
request certificate install /home/admin/<device>.crt
→ "Certificate Install Successful"
```

### Verification

Confirm that certificate status shows "Installed" on all nodes.

```
vBond# show control local-properties | include certificate-status
certificate-status                Installed
```

**【日本語サマリ】**<BR>
EVE-NG上でOpenSSLによりRoot CAを手動作成。
4台に対してSCP転送→CSR生成→署名→インストールを実施。<BR>
vManageが自動化している処理を手動で体験。

---

## 📝 Whitelist Registration (Manual, No vManage)

Without vManage, device serial numbers must be manually registered on the controllers. If a device is not registered, the connection attempt is rejected. The error codes seen in `show orchestrator connections-history` indicate the reason:

- **SERNTPRES** – Serial Number Not Present (device not in whitelist)
- **BIDNTVRFD** – Board ID Certificate Not Verified (device identity not trusted)

### On vBond: Register vSmart and vEdges

The vBond acts as the gatekeeper. First, register the vSmart controller using its serial number. Then register each vEdge using both chassis number and serial number.

```
vBond# request controller add serial-num 73349DF12462C9FB7D6FD80DCAAE178B4DAF7400 org-name Lab11

vBond# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vBond# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

### On vSmart: Register vEdges

The vSmart also maintains its own whitelist. Without these entries, vSmart rejects DTLS connections from vEdges with BIDNTVRFD.

```
vSmart# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vSmart# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

### Verification

Confirm that the registered devices appear in the orchestrator database with validity "valid".

```
vBond# show orchestrator valid-vsmarts
SERIAL NUMBER                             ORG
-------------------------------------------------
73349DF12462C9FB7D6FD80DCAAE178B4DAF7400  Lab11

vBond# show orchestrator valid-vedges
orchestrator valid-vedges 7DC18609-307B-4740-8BBF-D3680B46C41A
 serial-number                    5C5B3A628F9EA76750CA61FAE901CAC947937030
 validity                         valid
 org                              Lab11

orchestrator valid-vedges CD4DC9D3-8B58-434B-B17D-043359541538
 serial-number                    2E6BDAAFA60AD59836BC60240554BBBB402530D7
 validity                         valid
 org                              Lab11
```

**【日本語サマリ】**<BR>
vBondとvSmartにデバイスのシリアル番号を手動登録。<BR>
未登録だとSERNTPRES/BIDNTVRFDエラーで接続拒否される。

---

## ✅ Verification Results

### 1. MPLS Underlay

Verify that the MPLS backbone is operational. LDP neighbor should be in "Oper" state between PE routers. BGP on CE2 should show received prefixes (PfxRcd) from PE2, confirming that `allowas-in` is working — without it, PfxRcd would be 0 because CE1 and CE2 share the same AS 65000.

```
PE1# show mpls ldp neighbor
    Peer LDP Ident: 10.200.1.2:0; Local LDP Ident 10.200.1.1:0
        TCP connection: 10.200.1.2.43512 - 10.200.1.1.646
        State: Oper; Msgs sent/rcvd: ...; Downstream

CE2# show ip bgp summary
Neighbor        V   AS MsgRcvd MsgSent  TblVer  InQ OutQ Up/Down  State/PfxRcd
10.102.1.1      4  65001    32      32       7    0    0 00:19:10        4
```

### 2. Transport Reachability (vEdge → vBond via MPLS)

Test that vEdge10 at Site 2 can reach vBond at Site 1 through the MPLS VPN. This end-to-end path traverses: vEdge10 → CE2 → PE2 → MPLS core → PE1 → CE1 → vBond. Without this reachability, controller connections cannot be established.

```
vEdge10# ping vpn 0 10.1.2.2
PING 10.1.2.2 (10.1.2.2) 56(84) bytes of data.
64 bytes from 10.1.2.2: icmp_seq=1 ttl=60 time=26.8 ms
64 bytes from 10.1.2.2: icmp_seq=2 ttl=60 time=34.2 ms
--- 10.1.2.2 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss
```

### 3. Controller Connections

Verify that all DTLS control plane connections are established on vSmart. Expected: 4 connections — vBond ×2 (one per instance), vEdge02 (Site 1), and vEdge10 (Site 2). All should show state "up" with DTLS protocol.

```
vSmart# show control connections | tab
INSTANCE  PEER TYPE  SITE ID  SYSTEM IP    PROTOCOL  STATE  UPTIME
0         vbond      0        10.10.10.1   dtls      up     0:00:20:52
0         vedge      2        10.10.10.4   dtls      up     0:00:01:11
1         vedge      1        10.10.10.3   dtls      up     0:00:02:01
1         vbond      0        10.10.10.1   dtls      up     0:00:20:50
```

### 4. OMP Peers

Confirm that OMP peering is established between vSmart and both vEdges. OMP is the overlay routing protocol equivalent to BGP — it distributes service VPN routes through the vSmart controller.

```
vSmart# show omp peers | tab
PEER        TYPE   DOMAIN ID  SITE ID  STATE  UP TIME
10.10.10.3  vedge  1          1        up     0:00:02:57
10.10.10.4  vedge  1          2        up     0:00:02:08
```

### 5. BFD Sessions

Verify that BFD (Bidirectional Forwarding Detection) is running over the IPSec tunnel between vEdge02 and vEdge10. BFD monitors tunnel quality — latency, jitter, and packet loss — enabling SLA-based path selection.

```
vEdge02# show bfd sessions | tab
SRC IP    DST IP      PROTO  SYSTEM IP   SITE ID  STATE  UPTIME
10.1.1.2  10.200.2.2  ipsec  10.10.10.4  2        up     0:00:02:35
```

### 6. OMP Routes (Service VPN)

Confirm that VPN 1 service routes are exchanged between sites via OMP through vSmart. Each vEdge should see its own local route (FROM PEER = 0.0.0.0, status C,Red,R) and the remote site's route received from vSmart (FROM PEER = 10.10.10.2, status C,I,R). Status flags: **C** = Chosen, **I** = Installed in forwarding table, **R** = Resolved, **Red** = Redistributed from local.

```
vEdge02# show omp routes | tab
VPN  PREFIX           FROM PEER   STATUS  TLOC IP     COLOR    ENCAP
1    192.168.10.0/24  0.0.0.0     C,Red,R 10.10.10.3  default  ipsec
1    192.168.20.0/24  10.10.10.2  C,I,R   10.10.10.4  default  ipsec

vEdge10# show omp routes | tab
VPN  PREFIX           FROM PEER   STATUS  TLOC IP     COLOR    ENCAP
1    192.168.10.0/24  10.10.10.2  C,I,R   10.10.10.3  default  ipsec
1    192.168.20.0/24  0.0.0.0     C,Red,R 10.10.10.4  default  ipsec
```

**【日本語サマリ】**
MPLS Underlay → Transport到達性 → DTLS接続 → OMP Peer → BFD → OMPルート交換の順で検証し、全ステップ成功を確認。

---

## 🔧 Troubleshooting (Issues Encountered)

### Issue 1: BGP AS-Path Loop Between CEs

CE1 and CE2 both use AS 65000. When PE2 advertises CE1's routes to CE2, the AS path contains 65000, which CE2 rejects as a loop. This caused `show ip bgp summary` to show PfxRcd = 0 on CE2.

| | |
|---|---|
| **Symptom** | `CE2# show ip bgp summary` → PfxRcd = 0 |
| **Cause** | Same AS (65000) on both CEs; BGP loop prevention rejects routes containing own AS |
| **Fix** | `neighbor x.x.x.x allowas-in` on both CE1 and CE2 (under address-family ipv4) |

### Issue 2: Certificate Not Installed

After deploying initial configurations, `show control connections` returned no entries on all devices. Checking `show control local-properties` revealed that no certificate was installed — DTLS connections require valid certificates signed by a trusted CA.

| | |
|---|---|
| **Symptom** | `show control connections` → No entries found |
| **Diagnosis** | `show control local-properties` → certificate-status: Not-Installed |
| **Fix** | Full Enterprise Root CA workflow: generate CA → sign CSRs → install certificates on all nodes |

### Issue 3: Whitelist Not Configured (SERNTPRES / BIDNTVRFD)

Even after installing certificates, controller connections still failed. Checking `show orchestrator connections-history` on vBond revealed SERNTPRES and BIDNTVRFD errors, indicating that the connecting devices were not in the whitelist. In a vManage environment, this synchronization happens automatically.

| | |
|---|---|
| **Symptom** | `show orchestrator connections-history` → SERNTPRES / BIDNTVRFD errors |
| **Cause** | Without vManage, device serial numbers are not synced to vBond/vSmart automatically |
| **Fix** | `request controller add` (for vSmart) and `request vedge add` (for vEdges) on both vBond and vSmart |

### Issue 4: Empty OMP Routes

After all control connections came up, `show omp routes` returned empty on both vEdges. OMP only advertises routes from Service VPNs (VPN 1+), not from Transport VPN 0. Since no VPN 1 interface existed, there were no routes to advertise.

| | |
|---|---|
| **Symptom** | `show omp routes` → empty |
| **Cause** | No Service VPN (VPN 1) configured; OMP does not advertise VPN 0 transport routes |
| **Fix** | Create VPN 1 with loopback interface (physical LAN interface not connected in EVE-NG) |

**【日本語サマリ】**<BR>
下記で解決。<BR>
BGP同一ASループ→allowas-in<BR>
証明書未インストール→Root CA手動構築<BR>
ホワイトリスト未登録→request vedge add<BR>
OMPルート空→VPN 1作成**<BR>


---

## 🛠️ Lab Environment

| Component | Detail |
|---|---|
| **Platform** | EVE-NG Pro on PC (32GB RAM) |
| **Viptela OS** | 20.7.1 |
| **CE/PE** | Cisco IOL (IOS 15.x) |
| **vManage** | Not used (16GB RAM requirement exceeds lab capacity) |

---

## 📚 Key Takeaways

1. **Controller-based vs Appliance-based SD-WAN**: Viptela separates control (vSmart), orchestration (vBond), and data (vEdge) planes. FortiGate consolidates everything in a single appliance. The tradeoff is complexity vs scalability — Viptela can push policy changes to 100+ sites from one vSmart.

2. **vManage automates critical steps**: Certificate distribution, serial number synchronization, and template deployment are all manual without vManage. This lab exposes what happens "under the hood."

3. **OMP ≈ BGP for SD-WAN**: OMP is the overlay routing protocol, distributing service VPN routes through vSmart. It does not carry user data — IPSec tunnels handle that.

4. **Underlay independence**: The MPLS underlay (CEF + label switching) transports IPSec-encapsulated overlay packets. The overlay and underlay are logically separate but physically share the same infrastructure.

**【日本語サマリ】**<BR>
Viptelaはコントローラ分離型でスケールに有利。<BR>
vManageなし構築で証明書・ホワイトリストの内部動作を理解。<BR>
OMPはBGP相当の制御プレーンプロトコル。

---

## 🔗 Related Repositories

- [Enterprise-SP](https://github.com/mikio-abe/Enterprise-SP) – MPLS L3VPN underlay configuration
- [SASE-ZeroTrust](https://github.com/mikio-abe/SASE-ZeroTrust) – Cloudflare Zero Trust integration
- [SD-WAN (FortiGate)](https://github.com/mikio-abe/SD-WAN) – FortiGate SD-WAN with brownout testing
- [Troubleshooting](https://github.com/mikio-abe/Troubleshooting) – Network troubleshooting methodology
