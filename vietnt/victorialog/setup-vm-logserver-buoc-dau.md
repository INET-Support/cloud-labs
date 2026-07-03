# Setup VM Log Server — Hướng dẫn từ A→Z

Hướng dẫn dựng VM Ubuntu 24.04 làm log server cho stack RAG (VictoriaLogs + Qdrant + Postgres + Vector + Caddy).

Target: VM `192.168.122.53` trên vSphere lab, role log server cho 3 demo server (`122.50`, `122.51`, `122.52`).

---

## 0. Yêu cầu trước

- vSphere access tới host `202.92.7.202` (hoặc tương đương)
- ISO Ubuntu 24.04 Server LTS đã upload lên datastore
- Subnet lab `192.168.122.0/24`, gateway giả định `192.168.122.1`
- Máy dev (Windows) có route tới subnet lab (VPN hoặc cùng LAN)

---

## 1. Tạo VM trên vSphere

### 1.1 New Virtual Machine wizard

| Bước | Giá trị |
|---|---|
| Creation type | Create a new virtual machine |
| Name | `vietnt_ubuntu_24_04_192.168.122.53` |
| Folder | `iNET_LABS` |
| Compute resource | Host `202.92.7.202` |
| Storage | `datastore1 (2)` — **bắt buộc SSD/NVMe**, không HDD |
| Compatibility | ESXi 7.0 U2 and later (VM version 19) |
| Guest OS | Linux → Ubuntu Linux (64-bit) |

### 1.2 Customize hardware (QUAN TRỌNG)

| Mục | Giá trị | Lý do |
|---|---|---|
| CPU | **8 vCPU** | Đủ PoC + 50-100 server |
| Cores per Socket | **8** (1 socket) | Tránh vNUMA giả với VM ≤ 8 vCPU |
| Enable CPU Hot Add | ✅ tick | Scale CPU sau không reboot |
| Memory | **32 GB** | VL + Qdrant cache + headroom |
| New Hard disk | **1 TB**, Thick Eager Zeroed, SCSI(0:0), Persistent | I/O performance |
| SCSI controller | **VMware Paravirtual** (KHÔNG LSI Logic) | Throughput cao gấp 2-3 lần |
| Network adapter | **VMXNET 3** (KHÔNG E1000) + network `prv1622` | 10 Gbps virtual, low CPU |
| CD/DVD | Datastore ISO → chọn Ubuntu 24.04 LTS ISO | Boot installer |

### 1.3 Verify summary trước Finish

```
CPUs:          8
Memory:        32 GB
NICs:          1 (VMXNET 3, prv1622)
SCSI:          VMware Paravirtual
Hard disk:     1 TB, SCSI(0:0), Dependent
Compatibility: ESXi 7.0 U2
```

Click Finish → Power On VM.

---

## 2. Cài Ubuntu 24.04 Server

### 2.1 Boot installer

- Open vSphere Web Console → Power On → boot từ ISO
- Chọn ngôn ngữ: English
- Keyboard: English (US)
- Installation type: Ubuntu Server (minimized hoặc full đều OK)
- Network: để DHCP tạm thời, sẽ đổi static sau khi cài

### 2.2 Storage configuration — chia ổ thủ công

**KHÔNG dùng LVM** (KISS, dễ resize từ vSphere snapshot).

Layout đề xuất (disk 1 TB):

| # | Mount | Size | FS | Mục đích |
|---|---|---|---|---|
| 1 | (BIOS grub spacer) | 1 MB | — | Boot loader (installer tự tạo) |
| 2 | swap | 4 GB | swap | Emergency RAM |
| 3 | `/` | 100 GB | ext4 | OS + Docker images |
| 4 | `/opt/ragstack/data` | ~919 GB | ext4 | VL + Qdrant + Postgres data |

**Thao tác trên installer subiquity:**

1. `[ /dev/sda local disk 1.000T ]` → Enter → **Use As Boot Device**
2. `free space` → Add GPT partition → Size `4G`, Format `swap`
3. `free space` → Add GPT partition → Size `100G`, Format `ext4`, Mount `/`
4. `free space` → Add GPT partition → Size để trống (dùng hết), Format `ext4`, Mount **Other** → gõ `/opt/ragstack/data`
5. Verify FILE SYSTEM SUMMARY có đủ 4 partition → **Done** → Continue

### 2.3 Profile setup

| Field | Giá trị |
|---|---|
| Your name | `vietnt` |
| Server name | `logserver-01` |
| Pick username | `vietnt` |
| Password | (set password mạnh) |

### 2.4 SSH setup

✅ **Tick "Install OpenSSH server"** — bắt buộc, để rsync code lên VM sau.

Không import key qua GitHub (sẽ add manual sau).

### 2.5 Featured Server Snaps

Bỏ qua hết (không cần snap). Tiếp tục → Install → Reboot.

Sau reboot, eject ISO khỏi CD/DVD trong vSphere VM settings.

---

## 3. Cấu hình mạng tĩnh

Login vào VM qua vSphere console (chưa SSH được vì IP chưa cố định).

### 3.1 Xác định tên NIC

```bash
ip -br link
# lo               UNKNOWN
# ens192           UP             00:50:56:xx:xx:xx
```

Ghi nhớ tên NIC (giả sử `ens192`).

### 3.2 Disable cloud-init network override

Quan trọng — không làm thì netplan sẽ bị overwrite mỗi reboot.

```bash
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg <<'EOF'
network: {config: disabled}
EOF
```

### 3.3 Netplan static IP

```bash
sudo tee /etc/netplan/50-cloud-init.yaml <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens192:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.122.53/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        addresses:
          - 192.168.122.1
          - 1.1.1.1
          - 8.8.8.8
        search:
          - lab.local
      mtu: 1500
EOF

sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

**Lưu ý**: sửa `ens192` thành tên NIC thật, `192.168.122.1` thành gateway thật.

### 3.4 Set hostname

```bash
sudo hostnamectl set-hostname logserver-01
```

### 3.5 `/etc/hosts` resolve tên nội bộ

```bash
sudo tee /etc/hosts <<'EOF'
127.0.0.1       localhost
127.0.1.1       logserver-01
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

192.168.122.53  logserver-01 logserver app.local
192.168.122.50  mail-01
192.168.122.51  srv-01
192.168.122.52  srv-02
EOF
```

### 3.6 Verify network

```bash
ip -br addr show ens192          # ens192  UP  192.168.122.53/24
ip route                          # default via 192.168.122.1
ping -c 2 192.168.122.1           # gateway OK
ping -c 2 8.8.8.8                 # internet ICMP
ping -c 2 google.com              # DNS resolve OK
curl -fI https://download.docker.com   # HTTPS outbound OK
ping -c 1 mail-01                 # resolve qua /etc/hosts
```

Nếu DNS fail:
```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
resolvectl status ens192
```

---

## 4. Cấu hình SSH

### 4.1 Generate key trên máy dev (Windows PowerShell)

```powershell
ssh-keygen -t ed25519 -C "vietnt@dev" -f $env:USERPROFILE\.ssh\id_ed25519
# Enter để skip passphrase

# Xem public key, COPY toàn bộ
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

### 4.2 Cài key lên VM (lần đầu dùng password)

Trên VM qua vSphere console (hoặc SSH password lần đầu nếu network đã thông):

```bash
# Tạo authorized_keys cho user vietnt
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
# PASTE public key, save
chmod 600 ~/.ssh/authorized_keys

# Cài key cho root (cho phép root SSH bằng key)
sudo mkdir -p /root/.ssh && sudo chmod 700 /root/.ssh
sudo cp ~/.ssh/authorized_keys /root/.ssh/
sudo chmod 600 /root/.ssh/authorized_keys
sudo chown -R root:root /root/.ssh
```

### 4.3 Hardening sshd_config

```bash
sudo tee /etc/ssh/sshd_config.d/99-ragstack.conf <<'EOF'
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
AllowUsers vietnt root
ClientAliveInterval 60
ClientAliveCountMax 3
MaxAuthTries 3
LoginGraceTime 30
UseDNS no
LogLevel VERBOSE
EOF

sudo sshd -t                 # validate config (no output = OK)
sudo systemctl restart ssh
```

### 4.4 UFW firewall

```bash
sudo ufw allow from 192.168.122.0/24 to any port 22 proto tcp comment "ssh from lab"
sudo ufw allow from 192.168.122.0/24 to any port 514 proto udp comment "syslog ingest"
sudo ufw allow from 192.168.122.0/24 to any port 6514 proto tcp comment "syslog tls"
sudo ufw allow from 192.168.122.0/24 to any port 80 proto tcp comment "http"
sudo ufw allow from 192.168.122.0/24 to any port 443 proto tcp comment "https"
sudo ufw --force enable
sudo ufw status
```

### 4.5 Fail2ban (chống brute force)

```bash
sudo apt update && sudo apt install -y fail2ban

sudo tee /etc/fail2ban/jail.d/sshd.local <<'EOF'
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 3600
findtime = 600
EOF

sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### 4.6 SSH client config trên máy dev

Tạo `~/.ssh/config` (Windows: `%USERPROFILE%\.ssh\config`):

```
Host logsrv
    HostName 192.168.122.53
    User root
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30

Host mail-01
    HostName 192.168.122.50
    User vietnt
    IdentityFile ~/.ssh/id_ed25519

Host srv-01
    HostName 192.168.122.51
    User vietnt
    IdentityFile ~/.ssh/id_ed25519

Host srv-02
    HostName 192.168.122.52
    User vietnt
    IdentityFile ~/.ssh/id_ed25519
```

### 4.7 Test từ máy dev

```powershell
ssh logsrv                            # vào root, không password
ssh vietnt@192.168.122.53             # vào user
ssh notexist@192.168.122.53           # phải reject (AllowUsers)
```

---

## 5. Verify hệ thống sẵn sàng

### 5.1 OS info

```bash
# Verify partition mount đúng
df -h
# /dev/sda3       100G  ...  / 
# /dev/sda4       919G  ...  /opt/ragstack/data

# RAM, CPU
free -h          # 32G total, 4G swap
nproc            # 8

# Verify driver VMware đúng
lsmod | grep vmw_pvscsi          # PVSCSI driver loaded
ethtool -i ens192 | grep driver  # driver: vmxnet3
```

### 5.2 NTP sync

```bash
timedatectl status
# System clock synchronized: yes
# NTP service: active
```

Nếu chưa sync:
```bash
sudo apt install -y chrony
sudo systemctl enable --now chronyd
```

### 5.3 Cập nhật OS

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y open-vm-tools curl wget rsync vim htop
sudo reboot          # reboot lần cuối để clean
```

### 5.4 Cấu hình sysctl tuning (tối ưu cho stack)

```bash
sudo tee /etc/sysctl.d/99-ragstack.conf <<'EOF'
fs.file-max = 1000000
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.ip_local_port_range = 1024 65535
vm.swappiness = 10
vm.max_map_count = 262144
fs.inotify.max_user_watches = 524288
EOF

sudo sysctl -p /etc/sysctl.d/99-ragstack.conf
```

```bash
sudo tee /etc/security/limits.d/99-ragstack.conf <<'EOF'
*  soft  nofile  1000000
*  hard  nofile  1000000
EOF
```

---

## 6. Set ownership thư mục data

```bash
sudo mkdir -p /opt/ragstack/data/{victorialogs,qdrant,postgres,redis,vector}
sudo chown -R vietnt:vietnt /opt/ragstack
```

---

## 7. Rsync code stack từ máy dev

Trên máy dev (Windows):

```powershell
# Nếu có rsync (qua WSL hoặc Git Bash)
rsync -av D:/Vietnt/Project/cloud-labs/vietnt/infra/ D:/Vietnt/Project/cloud-labs/vietnt/mockups/ logsrv:/opt/ragstack/

# Hoặc scp
scp -r D:/Vietnt/Project/cloud-labs/vietnt/infra/* D:/Vietnt/Project/cloud-labs/vietnt/mockups vietnt@192.168.122.53:/opt/ragstack/
```

---

## 8. Bước tiếp theo

Sau khi xong setup trên:

1. Cài Docker theo `infra/scripts/setup-log-server.sh`
2. Cấu hình `.env` với password thực
3. `docker compose up -d` (profile default: 5 service VL + Qdrant + Postgres + Vector + Caddy)
4. Verify ingest log từ 3 demo server theo `docs/poc-deployment-guide.md`

---

## Troubleshooting

| Triệu chứng | Nguyên nhân | Fix |
|---|---|---|
| `ssh: connection timed out` | Máy dev không cùng subnet | VPN/jump host vào subnet 192.168.122.0/24 |
| Netplan apply mất kết nối | Sai gateway/NIC | Dùng vSphere console rollback `/etc/netplan/50-cloud-init.yaml` |
| Sau reboot IP về DHCP | Cloud-init override | Đã apply `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` chưa? |
| `Permission denied (publickey)` | Key sai/perm sai | `chmod 600 ~/.ssh/authorized_keys`, check fingerprint |
| DNS fail (`Temporary failure`) | systemd-resolved | `sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf` |
| `sudo: unable to resolve host` | `/etc/hosts` thiếu hostname | Thêm dòng `127.0.1.1 logserver-01` |
| Fail2ban ban IP máy dev | Sai password 3 lần | `sudo fail2ban-client set sshd unbanip <IP_dev>` |
| Disk full nhanh | VL log retention dài | Giảm `VL_RETENTION=7d` trong `.env` |

---

## Checklist cuối

```
[ ] VM created đúng spec: 8 vCPU, 32GB RAM, 1TB NVMe, PVSCSI, VMXNET3
[ ] Ubuntu 24.04 installed, 4 partition (boot/swap/root/data)
[ ] Static IP 192.168.122.53 apply OK, gateway/DNS work
[ ] /etc/hosts có 4 dòng ragstack hosts
[ ] SSH key login work cho cả vietnt và root
[ ] Password SSH bị reject
[ ] AllowUsers chỉ vietnt+root
[ ] UFW enable + rule cho 22/514/6514/80/443 từ 192.168.122.0/24
[ ] Fail2ban active
[ ] NTP synchronized
[ ] sysctl tuning applied
[ ] /opt/ragstack/data mounted 919GB ext4, owned bởi vietnt
[ ] open-vm-tools installed
[ ] OS updated + reboot xong
[ ] Code stack rsync lên /opt/ragstack/
```

Khi tất cả ✅ → sang `docs/poc-deployment-guide.md` cài Docker và bring stack up.
