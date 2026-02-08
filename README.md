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

**【日本語サマリ】**

MPLS L3VPN上にCisco Viptela SD-WANオーバーレイを構築（vManageなし、CLI only）。
Enterprise Root CAの手動作成・署名、ホワイトリスト登録、OMP経路交換、IPSec/BFDデータプレーンまでの全工程をCLIで実施。
制御プレーン分離アーキテクチャ（vBond/vSmart/vEdge）の動作を検証。

---

## 🏗️ Architecture

### Topology

<img width="760" alt="image" src="https://github.com/user-attachments/assets/af04d91a-240b-432c-8089-b2c4f0ef13d3" />

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

**【日本語サマリ】**

上の比較表はFortiGate SD-WANとViptela SD-WANの機能対応を示している。両方ともIPSecでデータを暗号化するが、経路交換はFortiGateがBGP、ViptelaはOMP。FortiGateは1台完結型、Viptelaはコントローラ分離型（SDN）。
Layered Architectureの表はOverlay（SD-WAN層）とUnderlay（MPLS層）の役割分担。OMPで経路情報を配布し、IPSecトンネルで実データを転送する。
Controller Rolesの表はViptelaの4コンポーネントの役割。vBondが認証（門番）、vSmartが経路配布（頭脳）、vEdgeがデータ転送（手足）、vManageが管理GUI（本ラボでは未使用）。

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

**【日本語サマリ】**

Underlay表：既存MPLS網のIPアドレス。PE1-PE2間が/30、各PE-CE間も/30で接続。
Overlay表：Lab11で追加したViptela機器の接続。CE1からvEdge02/vBond/vSmartの3台に直接接続、CE2からvEdge10に接続。
System Parameters表：各Viptelaノードの論理パラメータ。System-IPはOMP上の識別子、Site-IDは拠点番号（コントローラ=1000、Site1=1、Site2=2）、VPN 512はCloud0経由の管理アクセス用IPアドレス。

---

## 🔐 Enterprise Root CA (Manual, No vManage)

Without vManage, certificate management must be done entirely via CLI and OpenSSL. This is the process that vManage normally automates.

### Procedure

```
# Step 1: Create Root CA on EVE-NG host
mkdir -p /root/CA && cd /root/CA
openssl genrsa -out CA.key 2048
openssl req -x509 -new -nodes -key CA.key -sha256 -days 3650 \
  -out CA.pem -subj "/C=JP/O=Lab11/CN=Lab11-Root-CA"

# Step 2: Transfer CA.pem to all devices via SCP (Cloud0/VPN512)
scp CA.pem admin@192.168.133.10:/home/admin/CA.pem   # vBond
scp CA.pem admin@192.168.133.11:/home/admin/CA.pem   # vSmart
scp CA.pem admin@192.168.133.12:/home/admin/CA.pem   # vEdge02
scp CA.pem admin@192.168.133.13:/home/admin/CA.pem   # vEdge10

# Step 3: Install Root CA on each device
request root-cert-chain install /home/admin/CA.pem

# Step 4: Generate CSR on each device
request csr upload /home/admin/<device>.csr
# → Enter organization-unit name: Lab11

# Step 5: Sign CSRs with Root CA on EVE-NG host
openssl x509 -req -in vBond.csr -CA CA.pem -CAkey CA.key \
  -CAcreateserial -out vBond.crt -days 3650 -sha256
# (repeat for vSmart, vEdge02, vEdge10)

# Step 6: Install signed certificates on each device
request certificate install /home/admin/<device>.crt
```

### Verification

```
vBond# show control local-properties | include certificate-status
certificate-status                Installed
```

**【日本語サマリ】**

Step 1: EVE-NGホスト上で`openssl`コマンドを使い、秘密鍵（CA.key）と自己署名Root CA証明書（CA.pem）を作成。
Step 2: 作成したCA.pemをSCPで4台のViptelaノード（VPN 512の管理IP宛）に転送。
Step 3: 各ノードで`request root-cert-chain install`を実行 → "Successfully installed the root certificate chain"が返る。
Step 4: 各ノードで`request csr upload`を実行 → CSR（証明書署名要求）ファイルが生成される。
Step 5: CSRをEVE-NGに回収し、`openssl x509 -req`で署名 → 署名済み証明書（.crt）が生成される。
Step 6: 署名済み証明書を各ノードに転送し、`request certificate install`を実行 → "Certificate Install Successful"が返る。
検証: `show control local-properties`で`certificate-status: Installed`を確認。

---

## 📝 Whitelist Registration (Manual, No vManage)

Without vManage, device serial numbers must be manually registered. Unregistered devices are rejected with `SERNTPRES` (Serial Number Not Present) or `BIDNTVRFD` (Board ID Not Verified).

### On vBond: Register vSmart and vEdges

```
vBond# request controller add serial-num 73349DF12462C9FB7D6FD80DCAAE178B4DAF7400 org-name Lab11

vBond# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vBond# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

### On vSmart: Register vEdges

```
vSmart# request vedge add chassis-num 7dc18609-307b-4740-8bbf-d3680b46c41a \
  serial-num 5C5B3A628F9EA76750CA61FAE901CAC947937030 org-name Lab11

vSmart# request vedge add chassis-num cd4dc9d3-8b58-434b-b17d-043359541538 \
  serial-num 2E6BDAAFA60AD59836BC60240554BBBB402530D7 org-name Lab11
```

### Verification

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

**【日本語サマリ】**

まずvBondで`request controller add`を実行し、vSmartのシリアル番号を登録。次に`request vedge add`でvEdge02/vEdge10のchassis番号とシリアル番号を登録。
同じくvSmartでも`request vedge add`でvEdge02/vEdge10を登録（vSmartも未登録vEdgeからの接続を拒否するため）。
検証: `show orchestrator valid-vsmarts`でvSmartが1件、`show orchestrator valid-vedges`でvEdgeが2件登録されており、validity: validであることを確認。
これを行わないと`show orchestrator connections-history`にSERNTPRES（シリアル未登録）やBIDNTVRFD（ボードID未検証）エラーが表示されて接続できない。

---

## ✅ Verification Results

### 1. MPLS Underlay

```
PE1# show mpls ldp neighbor
    Peer LDP Ident: 10.200.1.2:0; Local LDP Ident 10.200.1.1:0
        TCP connection: 10.200.1.2.43512 - 10.200.1.1.646
        State: Oper; Msgs sent/rcvd: ...; Downstream

CE2# show ip bgp summary
Neighbor        V   AS MsgRcvd MsgSent  TblVer  InQ OutQ Up/Down  State/PfxRcd
10.102.1.1      4  65001    32      32       7    0    0 00:19:10        4
```
> CE1/CE2 both use AS 65000. `allowas-in` is required on both CEs to accept MPLS-transported routes containing their own AS number.

### 2. Transport Reachability (vEdge → vBond via MPLS)

```
vEdge10# ping vpn 0 10.1.2.2
PING 10.1.2.2 (10.1.2.2) 56(84) bytes of data.
64 bytes from 10.1.2.2: icmp_seq=1 ttl=60 time=26.8 ms
64 bytes from 10.1.2.2: icmp_seq=2 ttl=60 time=34.2 ms
--- 10.1.2.2 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss
```
> Path: vEdge10 → CE2 → PE2 → [MPLS] → PE1 → CE1 → vBond

### 3. Controller Connections

```
vSmart# show control connections | tab
INSTANCE  PEER TYPE  SITE ID  SYSTEM IP    PROTOCOL  STATE  UPTIME
0         vbond      0        10.10.10.1   dtls      up     0:00:20:52
0         vedge      2        10.10.10.4   dtls      up     0:00:01:11
1         vedge      1        10.10.10.3   dtls      up     0:00:02:01
1         vbond      0        10.10.10.1   dtls      up     0:00:20:50
```
> All 4 connections established: vBond ×2, vEdge02, vEdge10

### 4. OMP Peers

```
vSmart# show omp peers | tab
PEER        TYPE   DOMAIN ID  SITE ID  STATE  UP TIME
10.10.10.3  vedge  1          1        up     0:00:02:57
10.10.10.4  vedge  1          2        up     0:00:02:08
```

### 5. BFD Sessions

```
vEdge02# show bfd sessions | tab
SRC IP    DST IP      PROTO  SYSTEM IP   SITE ID  STATE  UPTIME
10.1.1.2  10.200.2.2  ipsec  10.10.10.4  2        up     0:00:02:35
```
> IPSec-encapsulated BFD session between vEdge02 (Site 1) and vEdge10 (Site 2)

### 6. OMP Routes (Service VPN)

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
> Both sites exchange VPN 1 routes via OMP through vSmart.
> Status: **C** (Chosen), **I** (Installed), **R** (Resolved) = fully operational.

**【日本語サマリ】**

1. `show mpls ldp neighbor`を実行 → PE1-PE2間のLDPがState: Operで確立済み。`show ip bgp summary`でCE2のPfxRcd=4、つまりCE1側の4ルートをBGPで受信できている（`allowas-in`により同一AS65000のルートを受け入れ）。
2. vEdge10から`ping vpn 0 10.1.2.2`（vBond宛）を実行 → 6パケット全成功、RTT 26-34ms。MPLS VPN経由（CE2→PE2→PE1→CE1→vBond）でvEdge10からvBondに到達できることを確認。
3. vSmartで`show control connections`を実行 → vBond×2、vEdge02（Site1）、vEdge10（Site2）の4接続がすべてDTLSプロトコルでstate: upと表示。
4. vSmartで`show omp peers`を実行 → vEdge02（System-IP 10.10.10.3）とvEdge10（10.10.10.4）の2ピアがstate: upと表示。OMPネイバーが確立。
5. vEdge02で`show bfd sessions`を実行 → vEdge10（DST 10.200.2.2）とのIPSec上BFDセッションがstate: upと表示。トンネル品質監視が動作中。
6. vEdge02/vEdge10で`show omp routes`を実行 → VPN 1の192.168.10.0/24と192.168.20.0/24が双方向で表示。FROM PEERが0.0.0.0は自身のローカルルート、10.10.10.2（vSmart）はvSmart経由で受信したリモートルート。Status: C,I,R（選択済み・インストール済み・解決済み）= 完全動作。

---

## 🔧 Troubleshooting (Issues Encountered)

### Issue 1: BGP AS-Path Loop Between CEs

| | |
|---|---|
| **Symptom** | `CE2# show ip bgp summary` → PfxRcd = 0 |
| **Cause** | CE1 and CE2 share AS 65000. Routes via MPLS contain AS 65000, triggering BGP loop prevention on the receiving CE. |
| **Fix** | `neighbor x.x.x.x allowas-in` on both CE1 and CE2 (under address-family ipv4) |

### Issue 2: Certificate Not Installed

| | |
|---|---|
| **Symptom** | `show control connections` → No entries found |
| **Diagnosis** | `show control local-properties` → certificate-status: Not-Installed |
| **Fix** | Full Enterprise Root CA workflow: generate CA → sign CSRs → install certificates on all nodes |

### Issue 3: Whitelist Not Configured (SERNTPRES / BIDNTVRFD)

| | |
|---|---|
| **Symptom** | `show orchestrator connections-history` shows SERNTPRES and BIDNTVRFD errors |
| **Cause** | Without vManage, device serial numbers are not synced to vBond/vSmart automatically |
| **Fix** | `request controller add` (for vSmart) and `request vedge add` (for vEdges) on vBond. Also `request vedge add` on vSmart. |

### Issue 4: Empty OMP Routes

| | |
|---|---|
| **Symptom** | `show omp routes` → empty |
| **Cause** | No Service VPN (VPN 1) configured. OMP only advertises service VPN routes, not transport VPN 0. |
| **Fix** | Create VPN 1 with loopback interface (physical interface not connected in EVE-NG) |

**【日本語サマリ】**

Issue 1: `show ip bgp summary`でPfxRcd=0（ルート受信ゼロ） → CE1/CE2が同一AS65000で、MPLS経由ルートのASパスに自ASが含まれBGPループ防止で拒否されていた → `allowas-in`をCE両方に設定して解決。
Issue 2: `show control connections`が空 → `show control local-properties`でcertificate-status: Not-Installedと表示 → Enterprise Root CAを手動作成・署名・インストールして解決。
Issue 3: `show orchestrator connections-history`にSERNTPRES/BIDNTVRFDエラー → デバイスのシリアル番号がvBond/vSmartに未登録 → `request controller add`と`request vedge add`で手動登録して解決。
Issue 4: `show omp routes`が空 → VPN 1（サービスVPN）が未設定。OMPはVPN 0のルートは広告しない → VPN 1にloopbackインターフェースを作成して解決。

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

**【日本語サマリ】**

1. Viptelaはコントローラ分離型で、vSmart 1台からポリシーを100拠点に一括配布できる。FortiGateは1台完結型で設計がシンプルだがスケールに限界がある。
2. 証明書配布、シリアル番号同期、テンプレート展開はすべてvManageが自動化する処理。本ラボではvManageなしで手動実行し、その「裏側」を体験した。
3. OMPはBGPに相当するオーバーレイ経路配布プロトコル。`show omp routes`で確認できるのは経路情報のみで、実データはIPSecトンネルが運ぶ。
4. MPLS Underlay（CEF+ラベルスイッチング）がIPSecカプセルを運搬する。`show omp routes`のTLOC IPとENCAPがOverlay/Underlayの接続点。

---

## 🔗 Related Repositories

- [Enterprise-SP](https://github.com/mikio-abe/Enterprise-SP) – MPLS L3VPN underlay configuration
- [SASE-ZeroTrust](https://github.com/mikio-abe/SASE-ZeroTrust) – Cloudflare Zero Trust integration
- [SD-WAN (FortiGate)](https://github.com/mikio-abe/SD-WAN) – FortiGate SD-WAN with brownout testing
- [Troubleshooting](https://github.com/mikio-abe/Troubleshooting) – Network troubleshooting methodology
