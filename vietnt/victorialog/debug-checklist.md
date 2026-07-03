# Debug Checklist — VictoriaLogs + vmalert + Alertmanager + Telegram

> Mục đích: khi alert Telegram không hoạt động hoặc trễ, chạy theo thứ tự A → F, paste output để xác định stage hỏng.
>
> Pipeline: `client logger → rsyslog → VictoriaLogs → vmalert → Alertmanager → Telegram`

---

## A. CONFIG FILES — xem nội dung hiện tại

```bash
# A1. Rule LogsQL
sudo cat /etc/vmalert/rules/logs.yml
groups:
  - name: log-alerts
    interval: 2s
    type: vlogs
    rules:
      - alert: SSHFailedLoginSpike
        expr: '_msg:"Failed password" | stats count() as fails | filter fails:>5'
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "SSH brute-force"
          description: "{{ $value }} lần Failed password gần đây"

      - alert: AnyErrorLog
        expr: 'severity:error | stats count() as errs | filter errs:>0'
        for: 0m
        labels:
          severity: info
        annotations:
          summary: "{{ $value }} log ERROR"
          description: "Kiểm tra VMUI"

# A2. Systemd unit vmalert
sudo cat /etc/systemd/system/vmalert.service
[Unit]
   Description=vmalert (VictoriaLogs rules)
   After=network.target victorialogs.service

   [Service]
   ExecStart=/usr/local/bin/vmalert \
  -rule=/etc/vmalert/rules/*.yml \
  -datasource.url=http://127.0.0.1:9428 \
  -notifier.url=http://127.0.0.1:9093 \
  -httpListenAddr=127.0.0.1:8880 \
  -evaluationInterval=2s \
  -rule.evalDelay=0s


   Restart=always

   [Install]
   WantedBy=multi-user.target

# A3. Alertmanager config
sudo cat /etc/alertmanager/alertmanager.yml
route:
  receiver: telegram
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 1h

receivers:
  - name: telegram
    telegram_configs:
      - bot_token: '8259683021:AAEI-vauSOtU89pAIhkZU9_62ZchAzm-bxM'
        chat_id: -5539450855
        parse_mode: HTML
        message: |
          <b>🚨 {{ .CommonLabels.alertname }}</b>
          {{ range .Alerts }}
          • <b>severity</b>: {{ .Labels.severity }}
          • <b>summary</b>: {{ .Annotations.summary }}
          • <b>detail</b>: {{ .Annotations.description }}
          {{ end }}

# A4. Systemd unit alertmanager
sudo cat /etc/systemd/system/alertmanager.service
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

# A5. VictoriaLogs systemd
sudo cat /etc/systemd/system/victorialogs.service
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

# A6. rsyslog client config (chạy trên CLIENT 192.168.122.52)
sudo cat /etc/rsyslog.d/95-victorialogs.conf
batch.maxsize="50"
batch.format="jsonarray"
action.execOnlyEveryNthTime="0"
queue.dequeueBatchSize="50"
queue.timeoutShutdown="500"
```

---

## B. SERVICE STATUS

```bash
# B1. Service đang chạy?
sudo systemctl status victorialogs alertmanager vmalert --no-pager | head -40
● victorialogs.service - VictoriaLogs
     Loaded: loaded (/etc/systemd/system/victorialogs.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-06-18 10:13:34 +07; 1h 25min ago
   Main PID: 61796 (victoria-logs)
      Tasks: 39 (limit: 4600)
     Memory: 14.6M (peak: 35.9M)
        CPU: 11.704s
     CGroup: /system.slice/victorialogs.service
             └─61796 /usr/local/bin/victoria-logs -storageDataPath=/var/lib/victorialogs -httpListenAddr=0.0.0.0:9428 -retentionPeriod=30d -loggerTimezone=Asia/Ho_Chi_Minh

Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.102+0700        info        VictoriaMetrics@v1.131.0/lib/logger/flag.go:20          -loggerTimezone="Asia/Ho_Chi_Minh"
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.102+0700        info        VictoriaMetrics@v1.131.0/lib/logger/flag.go:20          -retentionPeriod="30d"
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.102+0700        info        VictoriaMetrics@v1.131.0/lib/logger/flag.go:20          -storageDataPath="/var/lib/victorialogs"
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.102+0700        info        VictoriaLogs/app/victoria-logs/main.go:43        starting VictoriaLogs at "[0.0.0.0:9428]"...
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.103+0700        info        VictoriaLogs/app/vlstorage/main.go:141        opening storage at -storageDataPath=/var/lib/victorialogs
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.105+0700        info        VictoriaMetrics@v1.131.0/lib/memory/memory.go:45        limiting caches to 2463237734 bytes, leaving 1642158490 bytes to the OS according to -memory.allowedPercent=60, system memory limit 4105396224 bytes
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.114+0700        info        VictoriaLogs/app/vlstorage/main.go:147        successfully opened storage in 0.012 seconds; smallParts: 17; bigParts: 0; smallPartBlocks: 182; bigPartBlocks: 0; smallPartRows: 20667; bigPartRows: 0; smallPartSize: 1286485 bytes; bigPartSize: 0 bytes
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.114+0700        info        VictoriaLogs/app/victoria-logs/main.go:55        started VictoriaLogs in 0.012 seconds; see https://docs.victoriametrics.com/victorialogs/
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.115+0700        info        VictoriaMetrics@v1.131.0/lib/httpserver/httpserver.go:145        started server at http://0.0.0.0:9428/
Jun 18 10:13:34 server victoria-logs[61796]: 2026-06-18T10:13:34.115+0700        info        VictoriaMetrics@v1.131.0/lib/httpserver/httpserver.go:147        pprof handlers are exposed at http://0.0.0.0:9428/debug/pprof/

● alertmanager.service - Alertmanager
     Loaded: loaded (/etc/systemd/system/alertmanager.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-06-18 11:07:05 +07; 31min ago
   Main PID: 63379 (alertmanager)
      Tasks: 10 (limit: 4600)
     Memory: 14.7M (peak: 17.2M)
        CPU: 2.694s
     CGroup: /system.slice/alertmanager.service
             └─63379 /usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager --web.listen-address=127.0.0.1:9093

Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.470Z caller=main.go:181 level=info msg="Starting Alertmanager" version="(version=0.27.0, branch=HEAD, revision=0aa3c2aad14cff039931923ab16b26b7481783b5)"
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.470Z caller=main.go:182 level=info build_context="(go=go1.21.7, platform=linux/amd64, user=root@22cd11f671e9, date=20240228-11:51:20, tags=netgo)"
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.471Z caller=cluster.go:186 level=info component=cluster msg="setting advertise address explicitly" addr=192.168.122.50 port=9094
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.473Z caller=cluster.go:683 level=info component=cluster msg="Waiting for gossip to settle..." interval=2s
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.509Z caller=coordinator.go:113 level=info component=configuration msg="Loading configuration file" file=/etc/alertmanager/alertmanager.yml
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.509Z caller=coordinator.go:126 level=info component=configuration msg="Completed loading of configuration file" file=/etc/alertmanager/alertmanager.yml
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.512Z caller=tls_config.go:313 level=info msg="Listening on" address=127.0.0.1:9093
Jun 18 11:07:05 server alertmanager[63379]: ts=2026-06-18T04:07:05.512Z caller=tls_config.go:316 level=info msg="TLS is disabled." http2=false address=127.0.0.1:9093
Jun 18 11:07:07 server alertmanager[63379]: ts=2026-06-18T04:07:07.474Z caller=cluster.go:708 level=info component=cluster msg="gossip not settled" polls=0 before=0 now=1 elapsed=2.000863636s

# B2. Process + flags thực tế
ps -ef | grep -E 'victoria-logs|vmalert|alertmanager' | grep -v grep
vietnt@server:/$ ps -ef | grep -E 'victoria-logs|vmalert|alertmanager' | grep -v grep
victori+   61796       1  0 10:13 ?        00:00:11 /usr/local/bin/victoria-logs -storageDataPath=/var/lib/victorialogs -httpListenAddr=0.0.0.0:9428 -retentionPeriod=30d -loggerTimezone=Asia/Ho_Chi_Minh
root       63379       1  0 11:07 ?        00:00:02 /usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager --web.listen-address=127.0.0.1:9093
root       63524       1  0 11:09 ?        00:00:01 /usr/local/bin/vmalert -rule=/etc/vmalert/rules/*.yml -datasource.url=http://127.0.0.1:9428 -notifier.url=http://127.0.0.1:9093 -httpListenAddr=127.0.0.1:8880 -evaluationInterval=2s -rule.evalDelay=0s

# B3. Port đang lắng nghe (9428=VL, 9093=AM, 8880=vmalert)
ss -tlnp | grep -E '9428|9093|8880'
vietnt@server:/$ ss -tlnp | grep -E '9428|9093|8880'
LISTEN 0      4096       127.0.0.1:8880      0.0.0.0:*          
LISTEN 0      4096         0.0.0.0:9428      0.0.0.0:*          
LISTEN 0      4096       127.0.0.1:9093      0.0.0.0:*

# B4. Version
victoria-logs --version 2>&1 | head -3
vmalert        --version 2>&1 | head -3
alertmanager   --version 2>&1 | head -3
victoria-logs-20251205-181256-tags-v1.40.0-0-ga9e8c173f4
vmalert-20250124-135113-tags-v1.110.0-0-g4cab63c6a8
alertmanager, version 0.27.0 (branch: HEAD, revision: 0aa3c2aad14cff039931923ab16b26b7481783b5)
  build user:       root@22cd11f671e9
  build date:       20240228-11:51:20
```

---

## C. DATA PATH — log có vào VL không?

```bash
# C1. Sinh log có marker (chạy trên CLIENT)
MARK="dbg-$(date +%s)"
echo "MARKER=$MARK"
for i in $(seq 1 10); do logger -t sshd "Failed password $MARK $i"; done

# C2. Trên SERVER, dùng MARK ở C1
sleep 2
curl -s 'http://127.0.0.1:9428/select/logsql/query' \
  --data-urlencode "query=_msg:$MARK | stats count() as n"

# C3. Test query giống rule
curl -s 'http://127.0.0.1:9428/select/logsql/query' \
  --data-urlencode 'query=_msg:"Failed password" _time:1m | stats count() as fails'
```

---

## D. VMALERT — rule + evaluation

```bash
# D1. Rule loaded + state hiện tại
curl -s http://127.0.0.1:8880/api/v1/rules | python3 -m json.tool

# D2. Alert đang firing
curl -s http://127.0.0.1:8880/api/v1/alerts | python3 -m json.tool

# D3. Log vmalert (lỗi query / push notifier)
sudo journalctl -u vmalert --since "10 minutes ago" --no-pager | tail -50
```

---

## E. ALERTMANAGER — nhận + gửi Telegram

```bash
# E1. AM đang giữ alert nào
curl -s http://127.0.0.1:9093/api/v2/alerts | python3 -m json.tool

# E2. AM config thực tế đang load
curl -s http://127.0.0.1:9093/api/v2/status | python3 -m json.tool | head -40

# E3. Log AM (lỗi gửi Telegram)
sudo journalctl -u alertmanager --since "10 minutes ago" --no-pager \
  | grep -iE 'telegram|notify|error|warn' | tail -30
```

---

## F. TELEGRAM — bot có hoạt động?

> Thay `<BOT_TOKEN>` và `<CHAT_ID>` tương ứng.

```bash
# F1. Bot info
curl -s "https://api.telegram.org/bot<BOT_TOKEN>/getMe" | python3 -m json.tool

# F2. Gửi trực tiếp không qua AM
curl -s "https://api.telegram.org/bot<BOT_TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>" \
  -d "text=direct-test-$(date +%s)"

# F3. Test alert qua AM (bypass vmalert)
curl -XPOST http://127.0.0.1:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"ManualE2E","severity":"info"},"annotations":{"summary":"manual","description":"end-to-end test"}}]'
sleep 8
sudo journalctl -u alertmanager --since "1 minute ago" --no-pager | tail -20
```

---

## G. Bảng phân tích kết quả

| Output bất thường | Lỗi nằm ở | Hành động |
|---|---|---|
| **C2 = 0** | Client không gửi được log | Kiểm tra rsyslog (A6) + firewall + port 9428 mở chưa |
| **C3 = 0** nhưng C2 > 0 | Field/query LogsQL sai | Sửa expr trong rule (A1), kiểm tra `_msg` / `app` field trong VMUI |
| **D1 query không khớp rule mới** | vmalert chưa restart hoặc rule file chưa save | `sudo systemctl restart vmalert` |
| **D1 lastSamples = 0** nhưng C3 > 5 | Time window trong rule sai | Thêm `_time:30s` hoặc `_time:1m` vào expr |
| **D1 interval = 60** trong khi YAML đã ghi 2s | File chưa save / vmalert đọc file khác | Verify path `-rule=...` trong A2 khớp A1 |
| **D2 firing** nhưng **E1 rỗng** | vmalert không push được lên AM | Sai `-notifier.url` trong A2 |
| **E1 có alert** nhưng E3 không có `Notify success` | AM không gửi Telegram | Token/chat_id/mạng — xem E3 |
| **F2 không nhận** | Bot/chat_id sai | Lấy lại chat_id từ `getUpdates` |
| **F2 nhận, F3 không nhận** | AM config Telegram sai | Sửa A3, restart AM |

---

## H. Các lỗi phổ biến đã gặp

| Triệu chứng | Nguyên nhân | Fix |
|---|---|---|
| `Temporary failure in name resolution` | `/etc/resolv.conf` là symlink hỏng | `sudo rm /etc/resolv.conf && echo "nameserver 8.8.8.8" \| sudo tee /etc/resolv.conf` |
| `flag provided but not defined: -ruleType` | vmalert không có flag này | Bỏ flag, dùng `type: vlogs` trong YAML group |
| `yaml: line 6: did not find expected key` | Indent sai trong alertmanager.yml | `route:` và `receivers:` phải cùng cột 0; con thụt 2 spaces |
| `chat not found` | Sai `chat_id` hoặc chưa add bot vào group | Lấy lại từ `getUpdates`, supergroup thêm `-100` prefix |
| `apache2-utils` 404 khi apt install | Apt cache cũ | `sudo apt update` rồi cài lại |
| Alert Telegram chậm 30-60s | `interval`, `group_wait` mặc định cao | Đặt `interval: 2s` (vmalert), `group_wait: 1s` (AM); chấp nhận floor ~3-5s |

---

## I. Độ trễ end-to-end realistic

```
client logger → rsyslog batch → POST VL → VL index → vmalert eval → AM group_wait → Telegram
   <50ms        0.1-2s          <0.5s     ~1s        ≤interval     ≥group_wait    <1s
```

- **Tốt nhất** với pipeline này: ~3-5s
- Nếu cần <1s: dùng `fail2ban` action gọi Telegram trực tiếp (bypass VL pipeline)
- Để giảm: `interval: 2s` + `evalDelay: 0s` + `group_wait: 1s` + `for: 0s` + rsyslog `batch.maxsize=50`

---

## J. Sửa lệnh sinh log đúng pattern rule

Nếu rule yêu cầu `_msg:"Failed password"` và `count > 5`:

```bash
# Trên CLIENT
for i in $(seq 1 10); do
  logger -t sshd "Failed password for invalid user fake from 1.2.3.4 port 22 ssh2"
done
```

Sau 3-5s phải thấy alert. Nếu không → quay lại mục C → D → E theo thứ tự.
