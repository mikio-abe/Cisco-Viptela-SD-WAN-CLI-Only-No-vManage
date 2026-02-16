# Cisco Viptela SD-WAN (CLI-Only, No vManage)

💡 This repository documents a hands-on Cisco SD-WAN (Viptela) lab environment built **without vManage**. By intentionally removing vManage from the topology, every automated process — certificate distribution, device whitelist synchronization, template deployment — must be performed manually via CLI. The goal is to verify control plane and data plane behavior through CLI outputs, exposing the mechanisms that vManage normally hides behind its GUI.

---

## 🔬 Overview

This lab builds a Viptela SD-WAN fabric on top of an existing MPLS L3VPN underlay, using only CLI (no vManage GUI). It demonstrates the full controller-based SD-WAN lifecycle:

* **Controller Separation** – vBond (orchestrator), vSmart (control plane), vEdge (data plane)
* **Enterprise Root CA** – Manual certificate creation, signing, and installation via OpenSSL
* **Whitelist Registration** – Manual device authentication (normally automated by vManage)
* **OMP Route Exchange** – Overlay route distribution through vSmart controller
* **IPSec Data Plane** – Encrypted site-to-site tunnels with BFD monitoring

**【日本語サマリ】**<br>
MPLS L3VPN上にViptela SD-WANオーバーレイをCLI onlyで構築しました。<br>
証明書・ホワイトリスト・OMP・IPSec/BFDまでの全工程をvManageなしで実施しています。<br>
制御プレーン分離アーキテクチャの動作を検証しました。

---

## 🏗️ Architecture & IP Addressing

### Topology

The diagram below shows the three-layer structure: Cloud0/VPN 512 (management), SD-WAN Overlay (vBond/vSmart/vEdge connected via CEs), and MPLS Underlay (PE-PE backbone). All Viptela nodes connect to Cloud0 for out-of-band management, while data plane traffic flows through the CE-PE MPLS infrastructure.

<img width="650" alt="image" src="https://github.com/user-attachments/assets/974109d7-2baf-4a03-8f1c-325aa576cb40" />
### Underlay (MPLS)

| Link | Subnet | Device A | Device B |
| --- | --- | --- | --- |
| PE1–PE2 | 10.200.1.0/30 | PE1 e0/1: .1 | PE2 e0/1: .2 |
| PE1–CE1 | 10.101.1.0/30 | PE1 e0/0: .1 | CE1 e0/0: .2 |
| PE2–CE2 | 10.102.1.0/30 | PE2 e0/0: .1 | CE2 e0/0: .2 |

### Overlay (Viptela → CE)

| Link | Subnet | Device A | Device B |
| --- | --- | --- | --- |
| CE1–vEdge02 | 10.1.1.0/30 | CE1 e0/1: .1 | vEdge02 ge0/0: .2 |
| CE1–vBond | 10.1.2.0/30 | CE1 e0/2: .1 | vBond ge0/0: .2 |
| CE1–vSmart | 10.1.3.0/30 | CE1 e0/3: .1 | vSmart eth0: .2 |
| CE2–vEdge10 | 10.200.2.0/24 | CE2 e0/1: .1 | vEdge10 ge0/0: .2 |

### CE/PE Interface Summary

| Device | Interface | Connected To | IP Address | Role |
| --- | --- | --- | --- | --- |
| **PE1** | e0/0 | CE1 e0/0 | 10.101.1.1/30 | PE-CE (VRF) |
|  | e0/1 | PE2 e0/1 | 10.200.1.1/30 | MPLS Core |
|  | Loopback0 | — | 1.1.1.1/32 | BGP Router-ID |
| **PE2** | e0/0 | CE2 e0/0 | 10.102.1.1/30 | PE-CE (VRF) |
|  | e0/1 | PE1 e0/1 | 10.200.1.2/30 | MPLS Core |
|  | Loopback0 | — | 2.2.2.2/32 | BGP Router-ID |
| **CE1** | e0/0 | PE1 e0/0 | 10.101.1.2/30 | Uplink to MPLS |
|  | e0/1 | vEdge02 ge0/0 | 10.1.1.1/30 | SD-WAN Site 1 |
|  | e0/2 | vBond ge0/0 | 10.1.2.1/30 | Orchestrator |
|  | e0/3 | vSmart eth0 | 10.1.3.1/30 | Controller |
| **CE2** | e0/0 | PE2 e0/0 | 10.102.1.2/30 | Uplink to MPLS |
|  | e0/1 | vEdge10 ge0/0 | 10.200.2.1/24 | SD-WAN Site 2 |

### Viptela System Parameters

| Device | System-IP | Site-ID | Org | Mgmt (VPN 512) |
| --- | --- | --- | --- | --- |
| vBond | 10.10.10.1 | 1000 | Lab11 | 192.168.133.10 |
| vSmart | 10.10.10.2 | 1000 | Lab11 | 192.168.133.11 |
| vEdge02 | 10.10.10.3 | 1 | Lab11 | 192.168.133.12 |
| vEdge10 | 10.10.10.4 | 2 | Lab11 | 192.168.133.13 |

> **Note:** All Viptela nodes are connected to Cloud0 (VPN 512) for out-of-band management. Certificate distribution via SCP uses these management IPs.

**【日本語サマリ】**<br>
3層構造で構成しています: Cloud0/VPN 512（管理）、SD-WAN Overlay（vBond/vSmart/vEdge）、MPLS Underlay（PE-PE）。<br>
VPN 0=Transport、VPN 1=Service、VPN 512=Management（Cloud0経由、証明書SCP転送に使用）です。<br>
全ViptelaノードはCloud0経由でOut-of-Band管理に接続しています。

---

## 🔀 Protocol Design

This section compares the architectural approaches of FortiGate and Viptela SD-WAN, and explains how the overlay and underlay layers interact. FortiGate uses an all-in-one appliance model, while Viptela separates control, orchestration, and data planes across dedicated components.

### Protocol Stack Comparison: FortiGate vs Viptela

| Function | FortiGate SD-WAN | Viptela SD-WAN |
| --- | --- | --- |
| **Path Identity** | SD-WAN Member (interface-based: MPLS-VPN, SASE-VPN) | TLOC: Transport Locator (System-IP + Color + Encap) |
| **Data Encryption** | IPSec | IPSec |
| **Route Exchange** | BGP / Static | OMP (via vSmart) |
| **Path Monitoring** | Health-check (ping/HTTP) | BFD (delay/jitter/loss) |
| **Path Selection** | SLA target (latency/jitter/loss) per SD-WAN rule | vSmart policy per TLOC color (app-route policy) |
| **Control Plane** | Embedded (single appliance) | DTLS (vEdge ↔ vBond/vSmart) |
| **Management** | FortiManager (optional) | vManage (optional) |
| **Design Philosophy** | All-in-one appliance | Controller-separated (SDN) |

> **Path Identity is the core of SD-WAN.** Both platforms define WAN paths as selectable members — FortiGate uses interface-based SD-WAN Members (e.g. `MPLS-VPN`, `SASE-VPN`), while Viptela uses TLOCs identified by `System-IP + Color + Encap`. The `show omp routes` output shows TLOC per route: `TLOC IP: 10.10.10.4, COLOR: default, ENCAP: ipsec`. In a dual-path design, each WAN link gets a different color (e.g. `mpls`, `biz-internet`), and vSmart selects the best TLOC per application based on BFD metrics — equivalent to FortiGate's SLA-based failover.

### Layered Architecture

| Layer | Control Plane (Route Info) | Data Plane (Packet Forwarding) |
| --- | --- | --- |
| **Overlay (SD-WAN)** | OMP | IPSec tunnel |
| **Underlay (MPLS)** | BGP / OSPF / LDP | CEF + MPLS label switching |

**Data flow:**

```
Site1 → vEdge02 →[IPSec]→ CE1 →[CEF]→ PE1 →[MPLS]→ PE2 →[CEF]→ CE2 →[IPSec]→ vEdge10 → Site2
```

### Viptela Controller Roles

| Component | Role | Analogy |
| --- | --- | --- |
| **vBond** | Authentication & orchestration (device whitelist, initial connection brokering) | Gatekeeper |
| **vSmart** | Control plane (OMP route distribution, policy enforcement) | Brain |
| **vManage** | Management plane (GUI, templates, monitoring) – *not used in this lab* | Dashboard |
| **vEdge** | Data plane (IPSec tunnels, packet forwarding) | Hands & feet |

**【日本語サマリ】**<br>
FortiGateは1台完結型、Viptelaはコントローラ分離型（SDN）です。<br>
パス識別はFortiGateがSD-WAN Member（インターフェースベース）、ViptelaがTLOC（System-IP + Color + Encap）です。<br>
Overlay（OMP/IPSec）とUnderlay（BGP/MPLS）の2層構造でデータを転送します。<br>
vSmartがBFDメトリクスに基づきTLOCを選択する仕組みは、FortiGateのSLAベースフェイルオーバーに相当します。

---

## 🔐 Enterprise Root CA (Manual, No vManage)

Without vManage, certificate management must be done entirely via CLI and OpenSSL. This is the process that vManage normally automates behind the scenes.

### Step 1: Create Root CA on EVE-NG host

Generate a 2048-bit RSA private key and a self-signed Root CA certificate valid for 10 years.

```
mkdir -p /root/CA && cd /root/CA
openssl genrsa -out CA.key 2048
openssl req -x509 -new -nodes -key CA.key -sha256 -days 3650 \
  -out CA.pem -subj "/C=JP/O=Lab11/CN=Lab11-Root-CA"
```

EVE-NGホスト上で2048bit RSA秘密鍵と自己署名Root CA証明書（有効期限10年）を生成しました。

### Step 2: Distribute Root CA to all Viptela nodes

Transfer the CA certificate to each device via SCP using the management network (VPN 512 / Cloud0).

```
scp CA.pem admin@192.168.133.10:/home/admin/CA.pem   # vBond
scp CA.pem admin@192.168.133.11:/home/admin/CA.pem   # vSmart
scp CA.pem admin@192.168.133.12:/home/admin/CA.pem   # vEdge02
scp CA.pem admin@192.168.133.13:/home/admin/CA.pem   # vEdge10
```

管理ネットワーク（VPN 512 / Cloud0）経由で4台全ノードにCA証明書をSCP転送しました。

### Step 3: Install Root CA on each device

Install the Root CA certificate chain so each device trusts certificates signed by this CA.

```
request root-cert-chain install /home/admin/CA.pem
→ "Successfully installed the root certificate chain"
```

各デバイスにRoot CA証明書チェーンをインストールし、このCAが署名した証明書を信頼するよう設定しました。

### Step 4: Generate CSR on each device

Generate a Certificate Signing Request on each node. Enter "Lab11" as the organization-unit name when prompted.

```
request csr upload /home/admin/<device>.csr
```

各ノードでCSR（Certificate Signing Request）を生成しました。組織名は「Lab11」を指定しています。

### Step 5: Sign CSRs on EVE-NG host

Retrieve the CSRs via SCP, then sign each one with the Root CA to produce device certificates.

```
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

各デバイスからCSRをSCPで回収し、Root CAで署名してデバイス証明書を発行しました。

### Step 6: Install signed certificates on each device

Transfer the signed certificates back to each device and install them.

```
scp vBond.crt admin@192.168.133.10:/home/admin/vBond.crt
# (repeat for vSmart, vEdge02, vEdge10)
```

```
request certificate install /home/admin/<device>.crt
→ "Certificate Install Successful"
```

署名済みデバイス証明書を各ノードに転送しインストールしました。

### Verification

Confirm that certificate status shows "Installed" on all nodes.

```
vBond# show control local-properties | include certificate-status
certificate-status                Installed
```

→ 全ノードでcertificate-statusが「Installed」であることを確認しました。

**【日本語サマリ】**<br>
EVE-NG上でOpenSSLによりRoot CAを手動作成し、4台に対してSCP転送→CSR生成→署名→インストールを実施しました。<br>
vManageが自動化している証明書管理の全工程を手動で体験しています。

---

## 📝 Whitelist Registration (Manual, No vManage)

Without vManage, device serial numbers must be manually registered on the controllers. If a device is not registered, the connection attempt is rejected. The error codes seen in `show orchestrator connections-history` indicate the reason:

* **SERNTPRES** – Serial Number Not Present (device not in whitelist)
* **BIDNTVRFD** – Board ID Certificate Not Verified (device identity not trusted)

### On vBond: Register vSmart and vEdges

The vBond acts as the gatekeeper. First, register the vSmart controller using its serial number. Then register each vEdge using both chassis number and serial number.

```
vBond# request controller add serial-num 73349DF12462C9FB7D6FD80DCAAE178B4DAF7400 org-name Lab11

vBond# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vBond# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

vBond上でvSmartのシリアル番号、vEdge02/vEdge10のシャーシ番号+シリアル番号を手動登録しました。

### On vSmart: Register vEdges

The vSmart also maintains its own whitelist. Without these entries, vSmart rejects DTLS connections from vEdges with BIDNTVRFD.

```
vSmart# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vSmart# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

vSmart側にも同じvEdgeの情報を登録しました。未登録だとBIDNTVRFDエラーでDTLS接続が拒否されます。

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

→ vBondのオーケストレータDBに全デバイスがvalidity「valid」で登録されていることを確認しました。

**【日本語サマリ】**<br>
vBondとvSmartにデバイスのシリアル番号を手動登録しました。<br>
未登録の場合はSERNTPRES/BIDNTVRFDエラーで接続が拒否されます。<br>
vManage環境では自動同期される処理を手動で実施しています。

---

## ✅ Verification Results

### Operational Verification: FortiGate vs Viptela

Both platforms provide CLI commands to monitor tunnel health, quality metrics, and path selection. The key difference is that FortiGate makes failover decisions locally, while Viptela delegates path selection to vSmart based on BFD metrics reported by vEdges.

| Verification | FortiGate | Viptela |
| --- | --- | --- |
| **Tunnel UP/DOWN** | `diagnose sys sdwan member` | `show bfd sessions` |
| **Quality Metrics** | `diagnose sys sdwan health-check status` | `show app-route statistics` |
| **Current Path** | `diagnose sys sdwan service` | `show omp routes` (active TLOC = C,I,R) |
| **Failover Trigger** | SLA threshold breach → local decision | app-route policy threshold → vSmart decision |
| **Decision Maker** | FortiGate itself | vSmart controller (centralized) |

> **Note:** This lab uses a single TLOC (default/ipsec), so `show app-route statistics` reports metrics but no failover occurs. To verify failover behavior equivalent to the [FortiGate SD-WAN brownout test](https://github.com/mikio-abe/SD-WAN), a second WAN link with a different color (e.g. `biz-internet`) and an `app-route policy` on vSmart would be required.

**【日本語サマリ】**<br>
FortiGateはローカルでフェイルオーバー判断しますが、ViptelaはvSmartがBFDメトリクスに基づき集中制御します。<br>
本ラボはシングルTLOC構成のため、フェイルオーバー検証には2本目のWANリンク（別Color）とapp-route policyが必要です。

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

→ LDPネイバーがOper状態、CE2のBGPでPfxRcd=4を確認しました。allowas-inにより同一AS間でのルート受信が正常に動作しています。

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

→ Site 2（vEdge10）からSite 1（vBond）へのMPLS VPN経由のエンドツーエンド疎通を確認しました。0%パケットロスです。

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

→ vSmart上でvBond×2、vEdge02、vEdge10の計4つのDTLS接続が全てup状態であることを確認しました。

### 4. OMP Peers

Confirm that OMP peering is established between vSmart and both vEdges. OMP is the overlay routing protocol equivalent to BGP — it distributes service VPN routes through the vSmart controller.

```
vSmart# show omp peers | tab
PEER        TYPE   DOMAIN ID  SITE ID  STATE  UP TIME
10.10.10.3  vedge  1          1        up     0:00:02:57
10.10.10.4  vedge  1          2        up     0:00:02:08
```

→ vSmartとvEdge02/vEdge10間のOMPピアリングが確立していることを確認しました。

### 5. BFD Sessions

Verify that BFD (Bidirectional Forwarding Detection) is running over the IPSec tunnel between vEdge02 and vEdge10. BFD monitors tunnel quality — latency, jitter, and packet loss — enabling SLA-based path selection.

```
vEdge02# show bfd sessions | tab
SRC IP    DST IP      PROTO  SYSTEM IP   SITE ID  STATE  UPTIME
10.1.1.2  10.200.2.2  ipsec  10.10.10.4  2        up     0:00:02:35
```

→ vEdge02-vEdge10間でIPSecトンネル上のBFDセッションがup状態であることを確認しました。

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

→ VPN 1のサービスルートがvSmart経由でサイト間交換されていることを確認しました。リモートルートのステータスがC,I,R（Chosen, Installed, Resolved）で正常です。

**【日本語サマリ】**<br>
MPLS Underlay → Transport到達性 → DTLS接続 → OMP Peer → BFD → OMPルート交換の順で検証し、全ステップの成功を確認しました。<br>
FortiGateはローカル判断、ViptelaはvSmartによる集中判断という違いも確認しています。

---

## 🔧 Troubleshooting (Issues Encountered)

### Issue 1: BGP AS-Path Loop Between CEs

CE1 and CE2 both use AS 65000. When PE2 advertises CE1's routes to CE2, the AS path contains 65000, which CE2 rejects as a loop. This caused `show ip bgp summary` to show PfxRcd = 0 on CE2.

|  |  |
| --- | --- |
| **Symptom** | `CE2# show ip bgp summary` → PfxRcd = 0 |
| **Cause** | Same AS (65000) on both CEs; BGP loop prevention rejects routes containing own AS |
| **Fix** | `neighbor x.x.x.x allowas-in` on both CE1 and CE2 (under address-family ipv4) |

CE1/CE2が同一AS 65000のため、BGPループ防止機能がルートを拒否しPfxRcd=0になりました。<br>
`allowas-in`で自ASを含むルートの受信を許可して解決しました。

---

### Issue 2: Certificate Not Installed

After deploying initial configurations, `show control connections` returned no entries on all devices. Checking `show control local-properties` revealed that no certificate was installed — DTLS connections require valid certificates signed by a trusted CA.

|  |  |
| --- | --- |
| **Symptom** | `show control connections` → No entries found |
| **Diagnosis** | `show control local-properties` → certificate-status: Not-Installed |
| **Fix** | Full Enterprise Root CA workflow: generate CA → sign CSRs → install certificates on all nodes |

証明書が未インストールのためDTLS接続が確立しませんでした。<br>
`show control local-properties`でcertificate-status: Not-Installedを確認し、Enterprise Root CAの全工程を実施して解決しました。

---

### Issue 3: Whitelist Not Configured (SERNTPRES / BIDNTVRFD)

Even after installing certificates, controller connections still failed. Checking `show orchestrator connections-history` on vBond revealed SERNTPRES and BIDNTVRFD errors, indicating that the connecting devices were not in the whitelist. In a vManage environment, this synchronization happens automatically.

|  |  |
| --- | --- |
| **Symptom** | `show orchestrator connections-history` → SERNTPRES / BIDNTVRFD errors |
| **Cause** | Without vManage, device serial numbers are not synced to vBond/vSmart automatically |
| **Fix** | `request controller add` (for vSmart) and `request vedge add` (for vEdges) on both vBond and vSmart |

証明書インストール後も接続が失敗しました。vBondの接続履歴にSERNTPRES/BIDNTVRFDエラーが表示されていました。<br>
vManageなし環境ではデバイスのシリアル番号が自動同期されないため、手動でホワイトリスト登録して解決しました。

---

### Issue 4: Empty OMP Routes

After all control connections came up, `show omp routes` returned empty on both vEdges. OMP only advertises routes from Service VPNs (VPN 1+), not from Transport VPN 0. Since no VPN 1 interface existed, there were no routes to advertise.

|  |  |
| --- | --- |
| **Symptom** | `show omp routes` → empty |
| **Cause** | No Service VPN (VPN 1) configured; OMP does not advertise VPN 0 transport routes |
| **Fix** | Create VPN 1 with loopback interface (physical LAN interface not connected in EVE-NG) |

コントローラ接続確立後もOMPルートが空でした。<br>
OMPはService VPN（VPN 1以上）のルートのみ広告し、Transport VPN 0は対象外です。<br>
VPN 1にLoopbackインターフェースを作成して解決しました。

---

## 🛠️ Lab Environment

| Component | Detail |
| --- | --- |
| **Platform** | EVE-NG Pro on PC (32GB RAM) |
| **Viptela OS** | 20.7.1 |
| **CE/PE** | Cisco IOL (IOS 15.x) |
| **vManage** | Not used (16GB RAM requirement exceeds lab capacity) |

**【日本語サマリ】**<br>
EVE-NG Pro上でViptela OS 20.7.1を使用しています。<br>
vManageはRAM 16GB要件がラボ環境を超過するため未使用です。

---

## 🔍 Quick Validation Sequence

After deployment, verify the SD-WAN fabric in this order. Each step depends on the previous one succeeding.

```
1. Underlay        : CE# show ip bgp summary         → PfxRcd > 0
2. Transport       : vEdge# ping vpn 0 <vBond IP>    → 0% packet loss
3. Controller join : vSmart# show control connections  → all peers "up"
4. OMP peering     : vSmart# show omp peers            → all vEdges "up"
5. BFD tunnel      : vEdge# show bfd sessions          → state "up"
6. Route exchange  : vEdge# show omp routes             → VPN 1 prefixes with status C,I,R
```

**【日本語サマリ】**<br>
デプロイ後の検証順序です。<br>
Underlay→Transport到達性→コントローラ接続→OMP→BFD→ルート交換の順で確認しました。

---

## 📂 Evidence Directory

CLI outputs are organized in the `evidence/` directory for easy reference during interviews or review.

```
evidence/
  ce2_show_ip_bgp_summary.txt
  vedge10_ping_vbond.txt
  vsmart_show_control_connections.txt
  vsmart_show_omp_peers.txt
  vedge02_show_bfd_sessions.txt
  vedge02_show_omp_routes.txt
  vedge10_show_omp_routes.txt
  vbond_show_orchestrator_valid_vedges.txt
  vbond_show_orchestrator_valid_vsmarts.txt
```

**【日本語サマリ】**<br>
CLI検証出力をevidence/ディレクトリに整理しました。面談時の参照用です。

---

## 📚 Key Takeaways

1. **Controller-based vs Appliance-based SD-WAN**: Viptela separates control (vSmart), orchestration (vBond), and data (vEdge) planes. FortiGate consolidates everything in a single appliance. The tradeoff is complexity vs scalability — Viptela can push policy changes to 100+ sites from one vSmart.
2. **vManage automates critical steps**: Certificate distribution, serial number synchronization, and template deployment are all manual without vManage. This lab exposes what happens "under the hood."
3. **OMP ≈ BGP for SD-WAN**: OMP is the overlay routing protocol, distributing service VPN routes through vSmart. It does not carry user data — IPSec tunnels handle that.
4. **Underlay independence**: The MPLS underlay (CEF + label switching) transports IPSec-encapsulated overlay packets. The overlay and underlay are logically separate but physically share the same infrastructure.

**【日本語サマリ】**<br>
1. Viptelaはコントローラ分離型でスケールに有利です。FortiGateは1台完結型でシンプルです。<br>
2. vManageなし構築により、証明書・ホワイトリストの内部動作を理解できました。<br>
3. OMPはBGP相当のオーバーレイルーティングプロトコルです。ユーザデータはIPSecトンネルで転送します。<br>
4. MPLS Underlayはオーバーレイから論理的に独立しており、レイヤ分離の設計思想を確認しました。

---

## 🔗 Related Repositories

* [Enterprise-SP](https://github.com/mikio-abe/Enterprise-SP) – MPLS L3VPN underlay configuration
* [SASE-ZeroTrust](https://github.com/mikio-abe/SASE-ZeroTrust) – Cloudflare Zero Trust integration
* [SD-WAN (FortiGate)](https://github.com/mikio-abe/SD-WAN) – FortiGate SD-WAN with brownout testing
* [NGFW-Palo-Alto](https://github.com/mikio-abe/NGFW-Palo-Alto) – PA-VM IPSec VPN + BGP over IPSec
* [Troubleshooting](https://github.com/mikio-abe/Troubleshooting) – Network troubleshooting methodology
