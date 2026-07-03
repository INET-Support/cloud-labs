# Hướng dẫn cài đặt – sử dụng – test VictoriaLogs (Lab 2 server)

> Mô hình lab:
> - **Server VictoriaLogs**: `192.168.122.50` (nhận log, lưu trữ, query)
> - **Client**: `192.168.122.52` (gửi log lên server)
>
> OS: **Ubuntu 22.04/24.04** (các lệnh dưới đây chỉ test trên Ubuntu, dùng `apt` + `systemd`).

---

## 0. Chuẩn bị

Trên **cả 2 máy**:

```bash
# kiểm tra kết nối
ping -c 2 192.168.122.50   # chạy trên client
ping -c 2 192.168.122.52   # chạy trên server

# đồng bộ giờ (BẮT BUỘC - log lệch giờ sẽ query sai)
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
sudo systemctl enable --now systemd-timesyncd  # hoặc chronyd / ntpd
timedatectl status
```

Mở port trên **server 192.168.122.50**:

```bash
# Cổng 9428 = HTTP API + UI của VictoriaLogs
sudo ufw allow from 192.168.122.0/24 to any port 9428 proto tcp
sudo ufw reload
```

> Nếu `ufw` chưa enable: `sudo ufw enable` (cẩn thận mở sẵn port SSH: `sudo ufw allow 22/tcp`).

---

## 1. Cài VictoriaLogs trên SERVER (192.168.122.50)

### 1.1. Tải binary mới nhất

```bash
# Kiểm tra version mới nhất tại: https://github.com/VictoriaMetrics/VictoriaLogs/releases
VL_VER="v1.40.0"   # thay bằng version mới nhất

cd /tmp
wget https://github.com/VictoriaMetrics/VictoriaLogs/releases/download/${VL_VER}/victoria-logs-linux-amd64-${VL_VER}.tar.gz
tar xzf victoria-logs-linux-amd64-${VL_VER}.tar.gz
sudo install -m 0755 victoria-logs-prod /usr/local/bin/victoria-logs
victoria-logs --version
```

### 1.2. Tạo user + thư mục dữ liệu

```bash
sudo useradd -r -s /usr/sbin/nologin victorialogs
sudo mkdir -p /var/lib/victorialogs
sudo chown -R victorialogs:victorialogs /var/lib/victorialogs
```

### 1.3. Tạo systemd service

```bash
sudo tee /etc/systemd/system/victorialogs.service > /dev/null <<'EOF'
[Unit]
Description=VictoriaLogs
After=network.target

[Service]
Type=simple
User=victorialogs
Group=victorialogs
ExecStart=/usr/local/bin/victoria-logs \
  -storageDataPath=/var/lib/victorialogs \
  -httpListenAddr=0.0.0.0:9428 \
  -retentionPeriod=30d \
  -loggerTimezone=Asia/Ho_Chi_Minh
Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now victorialogs
sudo systemctl status victorialogs --no-pager
```

### 1.4. Kiểm tra hoạt động

```bash
# trên server
curl -s http://localhost:9428/health           # → OK
curl -s http://localhost:9428/metrics | head   # metrics Prometheus

# từ máy ngoài (laptop / client)
curl -s http://192.168.122.50:9428/health
```

Mở web UI: <http://192.168.122.50:9428/select/vmui>

---

## 2. Cấu hình CLIENT (192.168.122.52) gửi log

Có 3 cách phổ biến – chọn 1 (khuyến nghị: **rsyslog** vì có sẵn trên mọi distro).

### 2.1. Cách A — rsyslog (khuyến nghị)

#### Bước 1: cài module HTTP

```bash
sudo apt update
sudo apt install -y rsyslog rsyslog-omhttp
```

#### Bước 2: tạo config gửi log

```bash
sudo tee /etc/rsyslog.d/95-victorialogs.conf > /dev/null <<'EOF'
module(load="omhttp")

template(name="VLJson" type="list" option.jsonf="on") {
    property(outname="_time"    name="timereported" dateFormat="rfc3339" format="jsonf")
    property(outname="host"     name="hostname"     format="jsonf")
    property(outname="app"      name="programname"  format="jsonf")
    property(outname="severity" name="syslogseverity-text" format="jsonf")
    property(outname="facility" name="syslogfacility-text" format="jsonf")
    property(outname="_msg"     name="msg"          format="jsonf")
}

action(
    type="omhttp"
    server="192.168.122.50"
    serverport="9428"
    restpath="insert/jsonline?_stream_fields=host,app&_time_field=_time&_msg_field=_msg"
    template="VLJson"
    batch="on"
    batch.maxsize="1000"
    queue.type="LinkedList"
    queue.filename="vl_queue"
    queue.maxdiskspace="500m"
    queue.saveonshutdown="on"
    action.resumeRetryCount="-1"
)
EOF

sudo systemctl restart rsyslog
sudo systemctl status rsyslog --no-pager
```

> `queue.*` giúp lưu log ra đĩa khi server VictoriaLogs tạm chết → không mất log.

#### Bước 3: sinh log thử

```bash
logger -t testapp "Hello VictoriaLogs from client 192.168.122.52"
logger -t myapp -p user.err "test ERROR message"
logger -t myapp -p user.info "test INFO message"
```

### 2.2. Cách B — Vector (nhẹ, mạnh hơn rsyslog)

```bash
curl -1sLf https://repositories.timber.io/public/vector/setup.deb.sh | sudo bash
sudo apt install -y vector
```

`/etc/vector/vector.yaml`:

```yaml
sources:
  journal:
    type: journald
    current_boot_only: true

transforms:
  enrich:
    type: remap
    inputs: [journal]
    source: |
      ._time = .timestamp
      .host  = .host
      .app   = ._SYSTEMD_UNIT || .SYSLOG_IDENTIFIER || "unknown"
      ._msg  = .message

sinks:
  victorialogs:
    type: http
    inputs: [enrich]
    uri: "http://192.168.122.50:9428/insert/jsonline?_stream_fields=host,app&_time_field=_time&_msg_field=_msg"
    encoding:
      codec: json
    framing:
      method: newline_delimited
    batch:
      max_events: 1000
```

```bash
sudo systemctl enable --now vector
sudo journalctl -u vector -f
```

### 2.3. Cách C — curl test nhanh (không cần agent)

```bash
echo '{"_time":"'"$(date -u +%FT%TZ)"'","_msg":"manual curl log","host":"client-52","app":"curl-test","severity":"info"}' \
| curl -X POST -H 'Content-Type: application/json' \
  --data-binary @- 'http://192.168.122.50:9428/insert/jsonline?_stream_fields=host,app'
```

---

## 3. Test & xác minh

### 3.1. Kiểm tra log đã vào (Web UI)

Mở: <http://192.168.122.50:9428/select/vmui>

- Khung **Query**: gõ `*` → bấm **Execute**.
- Chọn **Time range** = "Last 15 minutes".
- Phải thấy các log `testapp`, `myapp`… từ host `client-52` / `192.168.122.52`.

### 3.2. Test qua HTTP API (curl)

Chạy **trên server** hoặc bất kỳ máy nào trong LAN:

```bash
VL="http://192.168.122.50:9428"

# 1. Tất cả log 5 phút gần nhất
curl -s "$VL/select/logsql/query" --data-urlencode 'query=_time:5m *' | head

# 2. Lọc theo app
curl -s "$VL/select/logsql/query" --data-urlencode 'query=app:testapp'

# 3. Lọc theo host
curl -s "$VL/select/logsql/query" --data-urlencode 'query=host:client-52 _time:1h'

# 4. Đếm log theo app trong 1h
curl -s "$VL/select/logsql/query" \
  --data-urlencode 'query=_time:1h * | stats by (app) count()'

# 5. Tìm chuỗi "ERROR"
curl -s "$VL/select/logsql/query" --data-urlencode 'query=ERROR _time:1h'

# 6. Liệt kê các stream (tổ hợp host+app)
curl -s "$VL/select/logsql/streams" --data-urlencode 'query=_time:1h'

# 7. Liệt kê field có trong log
curl -s "$VL/select/logsql/field_names" --data-urlencode 'query=_time:1h'
```

### 3.3. Bộ test end-to-end (script)

Lưu thành `test-vl.sh` trên client:

```bash
#!/usr/bin/env bash
set -e
VL=http://192.168.122.50:9428

echo "[1] Health server..."
curl -fsS $VL/health && echo

echo "[2] Sinh 5 log qua rsyslog..."
for i in $(seq 1 5); do
  logger -t e2etest -p user.info "e2e test message #$i ts=$(date +%s)"
done
sleep 3

echo "[3] Query lại..."
curl -s "$VL/select/logsql/query" \
  --data-urlencode 'query=app:e2etest _time:5m' | tee /tmp/vl-result.json
COUNT=$(grep -c '"_msg"' /tmp/vl-result.json || true)
echo "→ Số log nhận được: $COUNT"

[[ $COUNT -ge 5 ]] && echo "✅ PASS" || { echo "❌ FAIL"; exit 1; }
```

```bash
chmod +x test-vl.sh && ./test-vl.sh
```

### 3.4. Test ingest qua curl trực tiếp

```bash
# Gửi 100 log JSON
for i in $(seq 1 100); do
  echo "{\"_time\":\"$(date -u +%FT%TZ)\",\"_msg\":\"bulk log $i\",\"host\":\"client-52\",\"app\":\"bulktest\"}"
done | curl -X POST -H 'Content-Type: application/json' \
  --data-binary @- 'http://192.168.122.50:9428/insert/jsonline?_stream_fields=host,app'

# Đếm lại
curl -s 'http://192.168.122.50:9428/select/logsql/query' \
  --data-urlencode 'query=app:bulktest | stats count()'
```

### 3.5. Kiểm tra trên server

```bash
# disk usage
sudo du -sh /var/lib/victorialogs/

# log của service
sudo journalctl -u victorialogs -n 50 --no-pager

# metric quan trọng
curl -s http://localhost:9428/metrics | grep -E 'vl_(rows_ingested|storage_disk)'
```

---

## 4. LogsQL — cheatsheet nhanh

| Mục đích | Query |
|---|---|
| Tất cả log | `*` |
| Tìm chuỗi | `error` |
| Field cụ thể | `app:nginx` |
| AND | `app:nginx status:500` |
| OR | `app:nginx OR app:apache` |
| NOT | `error NOT app:cron` |
| Regex trong msg | `_msg:~"timeout.*db"` |
| Theo host | `host:client-52` |
| Time range | `_time:5m error` |
| Đếm theo field | `error \| stats by (app) count()` |
| Top 10 host lỗi | `error \| stats by (host) count() \| sort by (count) desc \| limit 10` |
| Trích regex | `* \| extract "user=<user>"` |

---

## 5. Troubleshooting

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| `curl: connection refused` từ client | Firewall server / `httpListenAddr=127.0.0.1` | Mở port 9428 + đảm bảo `0.0.0.0:9428` |
| Không thấy log mới | rsyslog chưa restart / sai URL | `systemctl restart rsyslog`, kiểm tra `journalctl -u rsyslog` |
| Log thiếu field `_time` | Template rsyslog sai | Đảm bảo dùng `dateFormat="rfc3339"` |
| Disk tăng nhanh | Retention dài / log spam | Giảm `-retentionPeriod`, lọc bớt ở client |
| Query chậm | Không khai báo `_stream_fields` | Khai báo `host,app` ở URL ingest |
| `omhttp` không có | Chưa cài package | `sudo apt install rsyslog-omhttp` |

Log debug rsyslog:
```bash
sudo rsyslogd -dn 2>&1 | grep -i omhttp
```

---

## 6. Bước tiếp theo (gợi ý)

1. **Grafana** trên server (cùng host hoặc khác):
   ```bash
   sudo grafana-cli plugins install victoriametrics-logs-datasource
   ```
   Datasource URL: `http://192.168.122.50:9428`.

2. **Bảo mật — nginx reverse proxy + basic auth** (VictoriaLogs không có auth built-in, BẮT BUỘC nếu expose ra LAN/Internet):

   **Bước 1 — Bind VictoriaLogs về localhost:**
   ```bash
   # Sửa /etc/systemd/system/victorialogs.service:
   #   -httpListenAddr=127.0.0.1:9428
   sudo systemctl daemon-reload
   sudo systemctl restart victorialogs
   ```

   **Bước 2 — Cài nginx + tạo user/pass:**
   ```bash
   sudo apt install -y nginx apache2-utils
   sudo htpasswd -c /etc/nginx/.vl_htpasswd admin    # nhập password
   # Thêm user khác:  sudo htpasswd /etc/nginx/.vl_htpasswd user2
   ```

   **Bước 3 — Config nginx** `/etc/nginx/sites-available/victorialogs`:
   ```nginx
   server {
       listen 80;
       server_name 192.168.122.50;

       # Cho phép client gửi log không cần auth (chỉ insert)
       location /insert/ {
           proxy_pass http://127.0.0.1:9428;
           # Optional: giới hạn IP client
           allow 192.168.122.0/24;
           deny all;
       }

       # UI + query bắt buộc đăng nhập
       location / {
           auth_basic           "VictoriaLogs";
           auth_basic_user_file /etc/nginx/.vl_htpasswd;
           proxy_pass           http://127.0.0.1:9428;
           proxy_set_header     Host $host;
       }
   }
   ```

   ```bash
   sudo ln -s /etc/nginx/sites-available/victorialogs /etc/nginx/sites-enabled/
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo nginx -t && sudo systemctl reload nginx

   # Firewall: đổi 9428 → 80
   sudo ufw allow from 192.168.122.0/24 to any port 80 proto tcp
   sudo ufw delete allow from 192.168.122.0/24 to any port 9428 proto tcp
   ```

   **Bước 4 — Truy cập:**
   - Web UI: `http://192.168.122.50/select/vmui` → popup nhập `admin` / password.
   - API: `curl -u admin:pass http://192.168.122.50/select/logsql/query ...`
   - Client rsyslog vẫn POST đến `192.168.122.50:80/insert/jsonline...` (không cần auth, có ACL IP).

   **TLS (tùy chọn):** thêm `listen 443 ssl;` + self-signed cert (`openssl req ...`) hoặc Let's Encrypt nếu có domain.

3. **Cảnh báo qua Telegram (vmalert + Alertmanager)**

   Mô hình: `vmalert` định kỳ chạy LogsQL trên VictoriaLogs → khi match → đẩy alert lên `alertmanager` → gửi Telegram.

   **Bước 1 — Tạo bot Telegram:**
   - Chat với [@BotFather](https://t.me/BotFather) → `/newbot` → lấy `BOT_TOKEN`.
   - Chat với bot 1 câu bất kỳ → lấy `CHAT_ID`: `curl -s https://api.telegram.org/bot<TOKEN>/getUpdates | grep -o '"chat":{"id":[0-9-]*'`

   **Bước 2 — Cài Alertmanager:**
   ```bash
   AM_VER=0.27.0
   cd /tmp
   wget https://github.com/prometheus/alertmanager/releases/download/v${AM_VER}/alertmanager-${AM_VER}.linux-amd64.tar.gz
   tar xzf alertmanager-${AM_VER}.linux-amd64.tar.gz
   sudo install -m0755 alertmanager-${AM_VER}.linux-amd64/alertmanager /usr/local/bin/
   sudo install -m0755 alertmanager-${AM_VER}.linux-amd64/amtool       /usr/local/bin/
   sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
   ```

   `/etc/alertmanager/alertmanager.yml`:
   ```yaml
   route:
     receiver: telegram
     group_wait: 10s
     group_interval: 30s
     repeat_interval: 1h

   receivers:
     - name: telegram
       telegram_configs:
         - bot_token: 'XXXX:YYYY'        # BOT_TOKEN
           chat_id: -1001234567890       # CHAT_ID (số, có dấu trừ nếu là group)
           parse_mode: HTML
           message: |
             <b>🚨 {{ .CommonLabels.alertname }}</b>
             {{ range .Alerts }}
             • <b>severity</b>: {{ .Labels.severity }}
             • <b>summary</b>: {{ .Annotations.summary }}
             • <b>detail</b>: {{ .Annotations.description }}
             {{ end }}
   ```

   Systemd unit `/etc/systemd/system/alertmanager.service`:
   ```ini
   [Unit]
   Description=Alertmanager
   After=network.target

   [Service]
   ExecStart=/usr/local/bin/alertmanager \
     --config.file=/etc/alertmanager/alertmanager.yml \
     --storage.path=/var/lib/alertmanager \
     --web.listen-address=127.0.0.1:9093
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now alertmanager
   ```

   **Bước 3 — Cài vmalert:**
   ```bash
   VM_VER=v1.110.0
   cd /tmp
   wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/${VM_VER}/vmutils-linux-amd64-${VM_VER}.tar.gz
   tar xzf vmutils-linux-amd64-${VM_VER}.tar.gz
   sudo install -m0755 vmalert-prod /usr/local/bin/vmalert
   sudo mkdir -p /etc/vmalert/rules
   ```

   **Bước 4 — Viết rule LogsQL** `/etc/vmalert/rules/logs.yml`:
   ```yaml
   groups:
     - name: log-alerts
       interval: 1m
       type: vlogs                 # ← bắt buộc: báo group dùng LogsQL (không phải PromQL)
       rules:
         - alert: SSHFailedLoginSpike
           # LogsQL: đếm SSH thất bại 5 phút gần nhất
           expr: |
             app:sshd "Failed password" _time:5m | stats count() as fails | filter fails:>5
           for: 0m
           labels:
             severity: warning
           annotations:
             summary: "SSH brute-force trên {{ $labels.host }}"
             description: "{{ $value }} lần đăng nhập SSH thất bại trong 5 phút"

         - alert: AnyErrorLog
           expr: |
             _time:1m severity:error | stats count() as errs | filter errs:>0
           for: 0m
           labels:
             severity: info
           annotations:
             summary: "Có {{ $value }} log ERROR phút vừa rồi"
             description: "Kiểm tra VMUI để xem chi tiết"
   ```

   Systemd unit `/etc/systemd/system/vmalert.service`:
   ```ini
   [Unit]
   Description=vmalert (VictoriaLogs rules)
   After=network.target victorialogs.service

   [Service]
   ExecStart=/usr/local/bin/vmalert \
     -rule=/etc/vmalert/rules/*.yml \
     -datasource.url=http://127.0.0.1:9428 \
     -notifier.url=http://127.0.0.1:9093 \
     -httpListenAddr=127.0.0.1:8880 \
     -rule.evalDelay=10s
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

   > **Quan trọng:** `type: vlogs` đặt trong rule YAML (mức group) báo cho vmalert biết group này dùng LogsQL.
   > Yêu cầu vmalert ≥ v1.93. Check: `vmalert --version`. Nếu cũ hơn, tải bản v1.110+ từ release page.

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now vmalert
   sudo journalctl -u vmalert -f
   ```

   **Bước 5 — Test:**
   ```bash
   # Sinh log SSH failed giả trên client
   for i in $(seq 1 10); do
     logger -t sshd "Failed password for invalid user fake from 1.2.3.4 port 22 ssh2"
   done
   # Đợi 1-2 phút → Telegram phải nhận tin "🚨 SSHFailedLoginSpike"
   ```

   **Web UI vmalert:** `http://127.0.0.1:8880` (qua SSH tunnel: `ssh -L 8880:127.0.0.1:8880 user@192.168.122.50`) — xem alert đang FIRING/PENDING.

4. **Backup định kỳ**:
   ```bash
   curl -X POST http://localhost:9428/snapshot/create
   ```

---

## 7. Tóm tắt

```
[Client 192.168.122.52]                 [Server 192.168.122.50]
    rsyslog/Vector       ── HTTP :9428 ──▶   VictoriaLogs
                                                │
                                                ├── /select/vmui   (UI)
                                                ├── /select/logsql (API)
                                                └── /var/lib/victorialogs (data)
```

- Server: 1 binary + 1 systemd service + 1 port 9428.
- Client: rsyslog + 1 file config + restart.
- Test: `logger` → query `app:testapp` thấy log = OK.
