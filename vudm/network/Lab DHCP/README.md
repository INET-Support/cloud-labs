# 🖧 Lab DHCP Server với dnsmasq trên Ubuntu 24.04

> **Mục tiêu:** Dựng DHCP Server bằng dnsmasq, cấu hình Server làm Gateway NAT, bắt gói tin DORA bằng tcpdump/tshark, kiểm tra IP Conflict bằng ARP.

---

## 📋 Mục lục

- [Topology](#topology)
- [Môi trường thực hiện](#môi-trường-thực-hiện)
- [Kết quả đạt được](#kết-quả-đạt-được)
- [Chi tiết thực hiện](#chi-tiết-thực-hiện)
  - [1. Chuẩn bị VMware](#1-chuẩn-bị-vmware)
  - [2. Cấu hình DHCP Server](#2-cấu-hình-dhcp-server)
  - [3. Cấu hình NAT Gateway](#3-cấu-hình-nat-gateway)
  - [4. Cài đặt dnsmasq](#4-cài-đặt-dnsmasq)
  - [5. Test DHCP Client](#5-test-dhcp-client)
  - [6. Bắt & Phân tích gói tin DORA](#6-bắt--phân-tích-gói-tin-dora)
  - [7. Test IP Conflict](#7-test-ip-conflict)
- [Phân tích kết quả](#phân-tích-kết-quả)
- [Kết luận](#kết-luận)
- [Screenshots](#screenshots)

---

## Topology

```
┌──────────────────────────────────────────────────────────────┐
│                      VMware Workstation                      │
│                                                              │
│  ┌───────────────────────┐      ┌───────────────────────┐   │
│  │      DHCP Server      │      │      DHCP Client      │   │
│  │     Ubuntu 24.04      │      │     Ubuntu 24.04      │   │
│  │                       │      │                       │   │
│  │  ens33 ──► VMnet8     │      │  ens33 ──► VMnet1     │   │
│  │  192.168.186.132/24   │      │  192.168.100.106/24   │   │
│  │  (NAT → Internet)     │      │  (nhận từ DHCP)       │   │
│  │                       │      └──────────┬────────────┘   │
│  │  ens37 ──► VMnet1     │                 │                │
│  │  192.168.100.1/24     │                 │                │
│  │  (DHCP Server + GW)   │                 │                │
│  └──────────┬────────────┘                 │                │
│             └──────────── VMnet1 ──────────┘                │
│                       192.168.100.0/24                      │
│                       Host-only, DHCP tắt                   │
│             VMnet8 (NAT) ──────────────► Internet           │
└──────────────────────────────────────────────────────────────┘
```

---

## Môi trường thực hiện

| Thành phần | Chi tiết |
|---|---|
| Hypervisor | VMware Workstation |
| OS | Ubuntu 24.04 LTS |
| DHCP Server | dnsmasq |
| Công cụ bắt gói tin | tcpdump, tshark, arping |
| Mạng nội bộ | VMnet1 Host-only — `192.168.100.0/24` |
| Mạng ra internet | VMnet8 NAT |

### Thông tin IP

| Node | Card mạng | Địa chỉ IP | Vai trò |
|---|---|---|---|
| Server | ens33 | 192.168.186.132/24 | Uplink — NAT ra internet |
| Server | ens37 | 192.168.100.1/24 | DHCP Server + Default Gateway |
| Client | ens33 | 192.168.100.106/24 | Nhận IP động từ DHCP |
| DHCP Pool | — | 192.168.100.100 – .110 | Dải địa chỉ được phép cấp |

---

## Kết quả đạt được

- ✅ Dựng thành công DHCP Server bằng dnsmasq
- ✅ Client nhận IP động từ pool `192.168.100.100–110`
- ✅ Server làm Gateway NAT — Client ping được internet
- ✅ Bắt đủ 5 gói tin: Release → Discover → Offer → Request → ACK
- ✅ Thấy rõ MAC address và các field trong gói DHCP Discover
- ✅ Reproduce IP Conflict TH1: ARP probe khi đặt manual IP chưa cấp
- ✅ Reproduce IP Conflict TH2: Gratuitous ARP flapping khi trùng IP đã cấp

---

## Chi tiết thực hiện

### 1. Chuẩn bị VMware

**Tắt DHCP của VMnet1** trong Virtual Network Editor:
- VMnet1 (Host-only): tắt "Use local DHCP service" → Subnet `192.168.100.0/24`
- VMnet8 (NAT): giữ nguyên DHCP của VMware

> Tắt DHCP của VMnet1 để toàn bộ việc cấp IP cho Client đều do dnsmasq xử lý — tránh xung đột với DHCP mặc định của VMware.

![Cấu hình Virtual Network Editor](<../Báo cáo DHCP/Cấu hình network.png>)

**Cấu hình VM:**

| VM | NIC | VMnet |
|---|---|---|
| Server | NIC1 | VMnet8 (NAT) |
| Server | NIC2 | VMnet1 (Host-only) |
| Client | NIC1 | VMnet1 (Host-only) |

![Settings VM Server — 2 NIC](<../Báo cáo DHCP/Settings máy server với 2 card mạng.png>)

![Settings VM Client — 1 NIC Host-only](<../Báo cáo DHCP/Settings máy client.png>)

---

### 2. Cấu hình DHCP Server

Kiểm tra card mạng hiện tại trên Server:

![Card mạng DHCP Server](<../Báo cáo DHCP/Card mạng DHCP Server.png>)

Đặt IP tĩnh cho `ens37` — card nội bộ kết nối với Client:

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true          # xin IP từ VMware NAT
    ens37:
      addresses:
        - 192.168.100.1/24 # IP tĩnh — sẽ là default gateway của Client
      dhcp4: false
```

```bash
sudo netplan apply
```

![Cấu hình card mạng server](<../Báo cáo DHCP/Cấu hình card mạng server.png>)

![Server sau khi apply — ip addr show 2 card](<../Báo cáo DHCP/Card mạng server khi apply cấu hình mới.png>)

---

### 3. Cấu hình NAT Gateway

```bash
# Bật IP Forwarding — cho phép kernel forward packet giữa ens37 ↔ ens33
sudo sysctl -w net.ipv4.ip_forward=1
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p

# MASQUERADE: thay src IP của packet đi ra ens33 bằng IP của ens33
sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE

# FORWARD: cho phép packet từ mạng nội bộ ra ngoài
sudo iptables -A FORWARD -i ens37 -o ens33 -j ACCEPT

# FORWARD: chỉ cho phép reply packets về (stateful — ESTABLISHED/RELATED)
sudo iptables -A FORWARD -i ens33 -o ens37 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Lưu vĩnh viễn
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

> **Cơ chế MASQUERADE:** Khi packet từ `192.168.100.106` đi ra ens33, kernel thay src IP thành `192.168.186.132` và lưu mapping vào conntrack table. Khi reply về, kernel tra conntrack và dịch ngược lại — Client không cần biết gì về NAT.

![Bật IP Forwarding cho Server](<../Báo cáo DHCP/Tiến hành bật IP Forwarding cho server.png>)

![Cấu hình iptables NAT MASQUERADE](<../Báo cáo DHCP/Cấu hình iptables.png>)

---

### 4. Cài đặt dnsmasq

```bash
sudo apt install -y dnsmasq tcpdump tshark
```

Cấu hình `/etc/dnsmasq.conf`:

```ini
# Chỉ lắng nghe trên card nội bộ
interface=ens37
bind-interfaces

# DHCP pool: cấp từ .100 đến .110, lease time 12 giờ
dhcp-range=192.168.100.100,192.168.100.110,12h

# Option 3 — Default Gateway trả về cho Client
dhcp-option=option:router,192.168.100.1

# Option 6 — DNS servers trả về cho Client
dhcp-option=option:dns-server,8.8.8.8,1.1.1.1

# Reply NAK ngay cho lease không hợp lệ thay vì im lặng
dhcp-authoritative

# Ghi log từng giao dịch DHCP
log-dhcp
log-facility=/var/log/dnsmasq.log
```

```bash
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

![Cấu hình dnsmasq.conf](<../Báo cáo DHCP/cấu hình dnsmas.png>)

![dnsmasq active running sau khi restart](<../Báo cáo DHCP/DNSmasq sau khi restart.png>)

---

### 5. Test DHCP Client

```bash
sudo ip link set ens33 up
sudo dhcpcd ens33
```

**Output của dhcpcd:**
```
ens33: offered 192.168.100.106 from 192.168.100.1
ens33: leased 192.168.100.106 for 43200 seconds
ens33: adding default route via 192.168.100.1
```

![Client nhận IP 192.168.100.106 từ DHCP](<../Báo cáo DHCP/DHCP Client leased.png>)

![Client ping được internet qua NAT Gateway](<../Báo cáo DHCP/Client đã có thể ping ra ngoài.png>)

Kiểm tra bảng lease trên Server:

```bash
cat /var/lib/misc/dnsmasq.leases
# 1779590628  00:0c:29:04:70:61  192.168.100.106  *  ff:29:04:70:61:...
# │            │                  │                │   │
# │            │                  │                │   └─ Client Identifier (Option 61)
# │            │                  │                └───── Hostname (* = không có)
# │            │                  └────────────────────── IP đã cấp
# │            └───────────────────────────────────────── MAC address
# └────────────────────────────────────────────────────── Unix timestamp hết hạn lease
```

![Bảng lease MAC↔IP trên Server](<../Báo cáo DHCP/Kết quả lease ở server.png>)

---

### 6. Bắt & Phân tích gói tin DORA

**Cơ chế DORA:**

```
CLIENT                                         SERVER
  │──── 1. DISCOVER (src 0.0.0.0 → broadcast) ────►│  "Có DHCP server nào không?"
  │◄─── 2. OFFER    (src server  → client)    ──────│  "Tôi đề xuất IP này cho bạn"
  │──── 3. REQUEST  (src 0.0.0.0 → broadcast) ────►│  "Tôi chọn offer của bạn"
  │◄─── 4. ACK      (src server  → client)    ──────│  "Xác nhận, IP là của bạn 12h"
```

> **Tại sao DISCOVER và REQUEST đều dùng broadcast?** Client chưa có IP nên không thể thiết lập kết nối unicast. REQUEST vẫn broadcast để thông báo cho *tất cả* DHCP server trong mạng biết client đã chọn server nào — các server không được chọn sẽ giải phóng IP đã reserve.

```bash
# Server — bắt gói tin port 67 (server) và 68 (client)
sudo tcpdump -i ens37 -w /tmp/dhcp_dora.pcap port 67 or port 68
```

![Chạy tcpdump trên Server](<../Báo cáo DHCP/chạy tcpdump ở server.png>)

```bash
# Client — release rồi xin lại để trigger đủ 5 gói
sudo dhcpcd --release ens33
sudo dhcpcd ens33

# Phân tích bằng tshark
tshark -r /tmp/dhcp_dora.pcap -Y bootp
```

![Client release IP rồi xin cấp lại](<../Báo cáo DHCP/Release IP rồi xin cấp lại ở client.png>)

![Phía Server bắt được các gói tin](<../Báo cáo DHCP/Phía server bắt các gói tin .png>)

**5 gói tin bắt được:**

| # | Thời gian | Gói tin | Src | Dst |
|---|---|---|---|---|
| 1 | 0.000000 | DHCP Release  | 192.168.100.106 | 192.168.100.1   |
| 2 | 4.127262 | DHCP Discover | 0.0.0.0         | 255.255.255.255 |
| 3 | 7.139577 | DHCP Offer    | 192.168.100.1   | 192.168.100.106 |
| 4 | 7.141423 | DHCP Request  | 0.0.0.0         | 255.255.255.255 |
| 5 | 7.145851 | DHCP ACK      | 192.168.100.1   | 192.168.100.106 |

![Xem 5 gói tin DORA bằng tshark](<../Báo cáo DHCP/Xem các gói tin bắt được.png>)

---

#### Phân tích từng field trong gói tin

Mỗi gói DHCP có header cố định 236 byte (kế thừa từ BOOTP) + phần Options variable. Dưới đây là breakdown các field quan trọng trong từng gói:

---

**Gói 1 — DHCP Release**

| Field | Giá trị | Ý nghĩa |
|---|---|---|
| `op` | `1` (BOOTREQUEST) | Chiều đi: Client → Server |
| `xid` | `0xABCD1234` | Transaction ID — giống phiên lease cũ để server tra cứu |
| `ciaddr` | `192.168.100.106` | Client điền IP đang giữ — khác với DISCOVER (để `0.0.0.0`) |
| `chaddr` | `00:0c:29:04:70:61` | MAC của Client |
| Option `53` | `7` (RELEASE) | Loại message |
| Option `54` | `192.168.100.1` | Chỉ rõ server nào cần xóa lease |

---

**Gói 2 — DHCP Discover**

| Field | Giá trị | Ý nghĩa |
|---|---|---|
| `op` | `1` (BOOTREQUEST) | Chiều đi: Client → Server |
| `htype` | `1` | Hardware type = Ethernet |
| `hlen` | `6` | MAC address dài 6 byte |
| `hops` | `0` | Không qua relay agent |
| `xid` | `0x3903F326` | Transaction ID mới — random, dùng để ghép cặp request/reply |
| `flags` | `0x8000` | **Broadcast flag = 1** — yêu cầu server broadcast reply vì client chưa nhận được unicast |
| `ciaddr` | `0.0.0.0` | Chưa có IP |
| `yiaddr` | `0.0.0.0` | Chưa được cấp |
| `giaddr` | `0.0.0.0` | Không có relay agent |
| `chaddr` | `00:0c:29:04:70:61` | **MAC — định danh chính của Client trong toàn bộ DORA** |
| Option `53` | `1` (DISCOVER) | Loại message |
| Option `55` | `[1, 3, 6, 12, ...]` | Parameter Request List — Client "đặt hàng" subnet mask, router, DNS... |
| Option `61` | `01:00:0c:29:04:70:61` | Client Identifier: `01` (Ethernet) + MAC |

---

**Gói 3 — DHCP Offer**

| Field | Giá trị | Ý nghĩa |
|---|---|---|
| `op` | `2` (BOOTREPLY) | Chiều đi: Server → Client |
| `xid` | `0x3903F326` | **Phải khớp với xid của DISCOVER** — Client dùng để nhận ra reply cho mình |
| `yiaddr` | `192.168.100.106` | **IP server đề xuất cấp cho Client** |
| `siaddr` | `192.168.100.1` | IP của Server |
| `chaddr` | `00:0c:29:04:70:61` | Server ghi lại MAC để tạo lease |
| Option `53` | `2` (OFFER) | Loại message |
| Option `54` | `192.168.100.1` | Server Identifier — Client cần nhớ để chọn đúng server |
| Option `51` | `43200` | Lease time: **43200 giây = 12 giờ** |
| Option `1` | `255.255.255.0` | Subnet mask |
| Option `3` | `192.168.100.1` | Default gateway |
| Option `6` | `8.8.8.8, 1.1.1.1` | DNS servers |

> Server **tạm thời reserve** IP `.106` cho MAC này nhưng chưa commit vào lease table — nếu Client không REQUEST trong timeout, server giải phóng IP.

---

**Gói 4 — DHCP Request**

| Field | Giá trị | Ý nghĩa |
|---|---|---|
| `op` | `1` (BOOTREQUEST) | Chiều đi: Client → Server |
| `xid` | `0x3903F326` | Giữ nguyên xid của phiên |
| `flags` | `0x8000` | Vẫn broadcast |
| `ciaddr` | `0.0.0.0` | Client chưa chính thức sử dụng IP |
| Option `53` | `3` (REQUEST) | Loại message |
| Option `54` | `192.168.100.1` | **"Tôi chọn server này"** — broadcast để server khác biết và giải phóng IP đã reserve |
| Option `50` | `192.168.100.106` | IP Client muốn — echo lại IP từ OFFER |

---

**Gói 5 — DHCP ACK**

| Field | Giá trị | Ý nghĩa |
|---|---|---|
| `op` | `2` (BOOTREPLY) | Chiều đi: Server → Client |
| `xid` | `0x3903F326` | Khớp xid |
| `yiaddr` | `192.168.100.106` | Xác nhận IP cho Client |
| Option `53` | `5` (ACK) | Loại message |
| Option `51` | `43200` | **Lease time chính thức bắt đầu tính từ lúc này** |
| Option `58` | `21600` | T1 Renewal: sau 6h Client unicast gia hạn với server |
| Option `59` | `37800` | T2 Rebinding: sau 10.5h Client broadcast tìm bất kỳ server |
| Option `3` | `192.168.100.1` | Default gateway |
| Option `6` | `8.8.8.8, 1.1.1.1` | DNS servers |

> Sau khi nhận ACK, Client gán IP vào interface, thêm default route, cấu hình DNS — và commit lease. Đồng hồ T1, T2 bắt đầu chạy.

---

**MAC address trong gói DISCOVER (tshark output):**

```
Client MAC address: VMware_04:70:61 (00:0c:29:04:70:61)
```

![Xem thông tin MAC address trong gói DISCOVER](<../Báo cáo DHCP/Xem thông tin địa chỉ MAC.png>)

> **Tại sao MAC là định danh, không phải IP?** Trong DISCOVER, client chưa có IP. MAC là địa chỉ vật lý gắn cứng với card mạng — duy nhất trong LAN — là cách duy nhất để server nhận ra đây là ai và cấp IP nhất quán cho họ.

---

### 7. Test IP Conflict

> **Cơ chế kiểm tra conflict:** Trước khi sử dụng IP (dù từ DHCP hay đặt manual), OS thực hiện **ARP Probe** (RFC 5227) — broadcast một ARP Request đặc biệt với `Sender IP = 0.0.0.0` để hỏi xem có ai đang dùng IP đó không mà không gây ô nhiễm ARP cache của người khác.

---

#### TH1 — Đặt manual IP chưa được cấp (192.168.100.105)

```bash
# Server — bắt ARP để quan sát
sudo tcpdump -i ens37 arp -n
```

![Bắt ARP + DHCP trên Server](<../Báo cáo DHCP/Bắt ARP + DHCP.png>)

```bash
# Client — đặt manual IP .105 (nằm trong pool nhưng chưa được cấp)
sudo ip addr add 192.168.100.105/24 dev ens33
```

![Đặt IP manual chưa được cấp (192.168.100.105)](<../Báo cáo DHCP/Đặt IP chưa được cấp.png>)

**ARP Probe quan sát được:**

```
ARP Request (Probe):
  Sender MAC : 00:0c:29:04:70:61
  Sender IP  : 0.0.0.0            ← đặc trưng của Probe — chưa chính thức dùng IP
  Target MAC : 00:00:00:00:00:00
  Target IP  : 192.168.100.105    ← IP đang được kiểm tra
  Broadcast  → ff:ff:ff:ff:ff:ff

Không ai reply → IP an toàn → Client gán .105 vào interface
```

![ARP Probe trên Client](<../Báo cáo DHCP/ARPING trên client.png>)

```bash
# Xác nhận từ Server — arping kiểm tra ai đang giữ .105
sudo arping -I ens37 192.168.100.105 -c 5
# → Unicast reply from 192.168.100.105 [00:0C:29:04:70:61]
```

![ARPING từ Server kiểm tra conflict](<../Báo cáo DHCP/ARPING trên server kiểm tra conflict.png>)

> **Điểm đáng chú ý:** DHCP Server không biết `.105` đã bị đặt manual. Nếu Client trả `.106` và xin lại, server có thể cấp `.105` cho một client khác — dẫn đến conflict tiềm tàng. Đây là lý do enterprise dùng IPAM để track cả IP static.

---

#### TH2 — Đặt manual IP trùng IP đã cấp (192.168.100.106)

```bash
# Dọn dẹp TH1
sudo ip addr del 192.168.100.105/24 dev ens33
```

![Xóa IP để chuẩn bị kiểm tra conflict TH2](<../Báo cáo DHCP/Xóa IP để cbi kiểm tra conflict.png>)

```bash
# Server giả lập một máy khác claim cùng IP .106
sudo ip addr add 192.168.100.106/24 dev ens37
sudo arping -I ens37 -c 5 -s 192.168.100.106 192.168.100.106
```

**Gratuitous ARP gửi ra:**

```
ARP Request (Gratuitous):
  Sender MAC : [MAC của ens37 Server]
  Sender IP  : 192.168.100.106    ← claim IP này là của tôi
  Target IP  : 192.168.100.106    ← Sender IP = Target IP → dấu hiệu Gratuitous ARP
  Broadcast  → ff:ff:ff:ff:ff:ff

Tác động:
  Tất cả máy trong subnet nhận gói → cập nhật ARP cache: .106 → MAC mới
  Client đang dùng .106 nhận gói → phát hiện MAC trong ARP cache bị thay đổi
  → IP Conflict detected!
```

**Kết quả tcpdump:**
```
ARP Request who-has 192.168.100.106 tell 192.168.100.106  ← Gratuitous ARP
ARP Request who-has 192.168.100.106 tell 192.168.100.106  ← lặp lại → ARP flapping
```

![ARP Conflict — Gratuitous ARP flapping](<../Báo cáo DHCP/ARP Conflict.png>)

![Sau khi kiểm tra conflict](<../Báo cáo DHCP/Sau khi kiểm tra conflict.png>)

**Client fallback sang APIPA `169.254.91.30`:**

```bash
# Client phát hiện conflict → gửi DHCP DECLINE về server
# Server đánh dấu .106 là "conflicted", tạm xóa khỏi pool
# Client random IP trong 169.254.0.0/16 (RFC 3927 — link-local)
# ARP Probe kiểm tra IP mới → gán vào interface
```

![Client fallback sang APIPA sau conflict](<../Báo cáo DHCP/client xin IP sau khi conflict.png>)

> **APIPA (`169.254.0.0/16`)** là range IANA dành riêng cho link-local. Máy dùng APIPA chỉ liên lạc được trong cùng L2 segment — không có default route, không ra internet được.

---

## Phân tích kết quả

### Vai trò của MAC address xuyên suốt lab

| Giai đoạn | Field chứa MAC | Vai trò |
|---|---|---|
| DHCP Discover | `chaddr` + Option `61` | Định danh Client để server tạo lease |
| DHCP Offer / ACK | `chaddr` trong lease table | Đảm bảo cùng MAC → cùng IP mỗi lần |
| ARP Probe | Sender MAC | Kiểm tra IP conflict trước khi dùng |
| Gratuitous ARP | Sender MAC thay đổi | Dấu hiệu có thiết bị khác claim IP |

### So sánh hai tình huống IP Conflict

| | TH1 — Manual IP chưa cấp | TH2 — Trùng IP đã cấp |
|---|---|---|
| **ARP pattern** | Probe: `Sender IP = 0.0.0.0` | Gratuitous: `Sender IP = Target IP` |
| **Hậu quả** | Cảnh báo nhẹ, mạng ổn | ARP cache flapping, mất kết nối |
| **Client phản ứng** | Gán IP thành công | DHCP DECLINE → APIPA fallback |
| **Rủi ro** | Server không biết IP bị chiếm | Outage cho user đang dùng IP đó |

---

## Kết luận

1. **dnsmasq** xử lý đầy đủ DORA theo RFC 2131 — nhẹ, log chi tiết, phù hợp lab và SMB.
2. **`xid` (Transaction ID)** là cơ chế ghép cặp request/reply — Client và Server dùng chung một `xid` xuyên suốt một phiên DORA.
3. **`chaddr` (MAC address)** là định danh cốt lõi trong DHCP — từ Discover đến lease table. Muốn IP cố định theo máy thì bind `MAC → IP` bằng `dhcp-host` trong dnsmasq.
4. **ARP Probe** (`Sender IP = 0.0.0.0`) là bước kiểm tra conflict phía client — DHCP Server không thể thay thế vì server không biết real-time state của toàn mạng.
5. **IP Conflict trong production** nguy hiểm vì gây ARP flapping → switch liên tục update CAM table → packet loss. Giải pháp dứt điểm: IPAM + DHCP Snooping + Dynamic ARP Inspection (DAI) trên switch.

---

## Screenshots

| File | Nội dung |
|---|---|
| `Cấu hình network.png` | Virtual Network Editor — VMnet1 tắt DHCP |
| `Settings máy server với 2 card mạng.png` | VM Server — 2 NIC |
| `Settings máy client.png` | VM Client — 1 NIC Host-only |
| `Card mạng DHCP Server.png` | Server — card mạng ban đầu |
| `Cấu hình card mạng server.png` | Cấu hình netplan cho ens37 |
| `Card mạng server khi apply cấu hình mới.png` | Server — ip addr show 2 card sau apply |
| `Tiến hành bật IP Forwarding cho server.png` | Bật IP Forwarding |
| `Cấu hình iptables.png` | iptables NAT MASQUERADE + FORWARD rules |
| `cấu hình dnsmas.png` | Nội dung file dnsmasq.conf |
| `DNSmasq sau khi restart.png` | dnsmasq active running |
| `DHCP Client leased.png` | Client nhận IP 192.168.100.106 |
| `Client đã có thể ping ra ngoài.png` | Client ping internet qua NAT Gateway |
| `Kết quả lease ở server.png` | Bảng lease — giải thích 5 cột |
| `chạy tcpdump ở server.png` | Khởi động tcpdump bắt port 67/68 |
| `Release IP rồi xin cấp lại ở client.png` | Client release + renew trigger DORA |
| `Phía server bắt các gói tin .png` | Server bắt được gói tin |
| `Xem các gói tin bắt được.png` | 5 gói tin DORA — tshark output |
| `Xem thông tin địa chỉ MAC.png` | Field breakdown gói DISCOVER |
| `Bắt ARP + DHCP.png` | tcpdump bắt ARP trên Server |
| `Đặt IP chưa được cấp.png` | Client đặt manual IP 192.168.100.105 |
| `ARPING trên client.png` | ARP Probe — Sender IP = 0.0.0.0 |
| `ARPING trên server kiểm tra conflict.png` | arping xác nhận ai đang giữ .105 |
| `Xóa IP để cbi kiểm tra conflict.png` | Dọn IP, chuẩn bị test TH2 |
| `ARP Conflict.png` | Gratuitous ARP — Sender IP = Target IP |
| `Sau khi kiểm tra conflict.png` | Trạng thái mạng sau conflict |
| `client xin IP sau khi conflict.png` | Client fallback APIPA 169.254.91.30 |
