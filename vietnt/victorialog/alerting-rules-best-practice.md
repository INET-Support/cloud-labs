# Alerting Best Practice — vmalert + Alertmanager + Telegram

> Stack: `Vector → VictoriaLogs → vmalert → Alertmanager → Telegram`
>
> Mục tiêu: alert có ý nghĩa, không spam, phân tầng theo mức nghiêm trọng.

---

## 1. Recommended `/etc/vmalert/rules/logs.yml`

Phân theo **mức nghiêm trọng** + window phù hợp từng pattern.

```yaml
groups:
  # ============ CRITICAL — báo ngay, kêu liên tục ============
  - name: critical
    interval: 5s
    type: vlogs
    rules:
      - alert: HostSilent
        expr: '_time:5m host:client-01 | stats count() as n | filter n:==0'
        for: 1m
        labels: {severity: critical, team: ops}
        annotations:
          summary: "❌ HOST CHẾT — client-01 không gửi log 5 phút"
          description: "Vector trên client-01 có thể đã dừng hoặc host crash"

      - alert: KernelOOM
        expr: '_time:1m (_msg:"Out of memory" OR _msg:"oom-killer") | stats by (host) count() as n | filter n:>0'
        for: 0s
        labels: {severity: critical, team: ops}
        annotations:
          summary: "💥 OOM kill trên {{ $labels.host }}"
          description: "Process bị OOM killer giết — kiểm tra RAM"

      - alert: KernelPanic
        expr: '_time:1m (_msg:panic OR _msg:"BUG:" OR _msg:"general protection fault" OR _msg:segfault) | stats by (host) count() as n | filter n:>0'
        for: 0s
        labels: {severity: critical, team: ops}
        annotations:
          summary: "💀 Kernel panic / segfault ({{ $value }} log)"

      - alert: DiskFull
        expr: '_time:2m (_msg:"No space left" OR _msg:"disk full") | stats by (host) count() as n | filter n:>0'
        for: 0s
        labels: {severity: critical, team: ops}
        annotations:
          summary: "💾 Đĩa đầy trên {{ $labels.host }}"

  # ============ WARNING — security ============
  - name: security
    interval: 10s
    type: vlogs
    rules:
      - alert: SSHBruteForce
        expr: '_time:1m SYSLOG_IDENTIFIER:sshd _msg:"Failed password" | stats by (host) count() as n | filter n:>5'
        for: 0s
        labels: {severity: warning, team: sec}
        annotations:
          summary: "🔐 SSH brute-force {{ $labels.host }}: {{ $value }} fail/1m"

      - alert: SSHRootLogin
        expr: '_time:1m SYSLOG_IDENTIFIER:sshd _msg:"Accepted" _msg:"root" | stats by (host) count() as n | filter n:>0'
        for: 0s
        labels: {severity: warning, team: sec}
        annotations:
          summary: "👑 Root login SSH trên {{ $labels.host }}"

      - alert: SudoAbuse
        expr: '_time:5m SYSLOG_IDENTIFIER:sudo _msg:"authentication failure" | stats by (host) count() as n | filter n:>3'
        for: 0s
        labels: {severity: warning, team: sec}
        annotations:
          summary: "🛡️ Nhiều lần sudo fail {{ $labels.host }}: {{ $value }}/5m"

      - alert: NewUserCreated
        expr: '_time:5m (_msg:"new user" OR _msg:"useradd") | stats by (host) count() as n | filter n:>0'
        for: 0s
        labels: {severity: warning, team: sec}
        annotations:
          summary: "👤 User mới được tạo trên {{ $labels.host }}"

  # ============ WARNING — service ============
  - name: service-health
    interval: 30s
    type: vlogs
    rules:
      - alert: ServiceRestart
        expr: '_time:5m _msg:"Stopped" SYSLOG_IDENTIFIER:systemd | stats by (host) count() as n | filter n:>3'
        for: 0s
        labels: {severity: warning, team: ops}
        annotations:
          summary: "🔄 {{ $labels.host }}: nhiều service restart ({{ $value }}/5m)"

      - alert: ErrorSpike
        expr: '_time:1m PRIORITY:<="3" | stats by (host) count() as n | filter n:>20'
        for: 30s
        labels: {severity: warning, team: ops}
        annotations:
          summary: "⚠️ {{ $labels.host }}: {{ $value }} log ERROR/1m (ngưỡng 20)"

  # ============ INFO — theo dõi nhẹ ============
  - name: info
    interval: 60s
    type: vlogs
    rules:
      - alert: HighLogVolume
        expr: '_time:5m host:client-01 | stats count() as n | filter n:>5000'
        for: 2m
        labels: {severity: info, team: ops}
        annotations:
          summary: "📈 Volume log bất thường client-01: {{ $value }}/5m"
```

> **Tinh chỉnh:** ngưỡng (`>5`, `>20`, `>5000`) cần điều chỉnh sau 1 tuần observ baseline thực tế.

---

## 2. Chống spam Telegram — kiến trúc 5 lớp

```
[vmalert]  →  for:N   →  inhibit  →  group_by  →  repeat_interval  →  [Telegram]
   (rule)     (debounce)  (suppress) (gom alert)   (chu kỳ lặp)
```

### Lớp 1 — `for:` (debounce ở vmalert)

| Severity | `for:` | Lý do |
|---|---|---|
| critical | `0s` | Báo ngay, không chờ |
| warning | `30s` | Phải duy trì 30s mới fire (lọc nhiễu) |
| info | `2m` | Phải duy trì 2 phút |

### Lớp 2 — Inhibit rules (Alertmanager)

Khi critical xảy ra → mute warning/info cùng host (tránh nhiễu khi đang xử lý sự cố lớn).

```yaml
inhibit_rules:
  - source_matchers:
      - severity = critical
    target_matchers:
      - severity =~ "warning|info"
    equal: [host]
```

### Lớp 3 — Group_by gom alert + route phân tầng

```yaml
route:
  receiver: telegram-noop
  group_by: [alertname, host]
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 30m

  routes:
    # Critical đi route riêng, nhanh
    - matchers: [severity = critical]
      receiver: telegram-critical
      group_wait: 0s
      group_interval: 1m
      repeat_interval: 15m

    # Security đi nhóm khác
    - matchers: [team = sec]
      receiver: telegram-security
      group_wait: 5s
      group_interval: 5m
      repeat_interval: 1h

    # Info chỉ gửi 1 lần / vài giờ
    - matchers: [severity = info]
      receiver: telegram-info
      group_wait: 1m
      group_interval: 30m
      repeat_interval: 6h
```

### Lớp 4 — Multi-receiver (chia kênh Telegram)

```yaml
receivers:
  - name: telegram-critical
    telegram_configs:
      - bot_token: 'XXX'
        chat_id: -100xxx          # group "🚨 Critical"
        parse_mode: HTML
        message: |
          <b>🚨🚨 {{ .CommonLabels.alertname }}</b>
          {{ range .Alerts }}• {{ .Annotations.summary }}
          {{ end }}

  - name: telegram-security
    telegram_configs:
      - bot_token: 'XXX'
        chat_id: -100yyy          # group "🔐 Security"
        parse_mode: HTML
        message: |
          <b>🔐 {{ .CommonLabels.alertname }}</b>
          {{ range .Alerts }}• {{ .Annotations.summary }}
          {{ end }}

  - name: telegram-info
    telegram_configs:
      - bot_token: 'XXX'
        chat_id: -100zzz          # group "📊 Info Daily"
        parse_mode: HTML
        send_resolved: false      # info không cần báo resolved

  - name: telegram-noop
    telegram_configs:
      - bot_token: 'XXX'
        chat_id: -100zzz
        send_resolved: false
```

### Lớp 5 — Silence theo lịch / sự kiện

Mute info giờ ngủ:
```bash
amtool silence add severity=info \
  --duration=12h \
  --comment="đêm không cần info" \
  --alertmanager.url=http://127.0.0.1:9093
```

Mute trong lúc maintenance:
```bash
amtool silence add host=client-01 \
  --duration=2h \
  --comment="maintenance window"
```

---

## 3. Bảng tổng kết tham số chống spam

| Tham số | Cài đặt khuyến nghị | Ý nghĩa |
|---|---|---|
| Rule `for:` | `0s/30s/2m` theo severity | Debounce ở vmalert |
| `group_wait` | 0s / 5s / 10s / 1m theo severity | Đợi gom batch đầu tiên |
| `group_interval` | 1m / 5m / 30m | Đợi giữa các batch trong cùng group |
| `repeat_interval` | 15m / 1h / 6h | Gửi lại nếu vẫn firing |
| `inhibit_rules` | critical mute warning cùng host | Tránh ngập khi sự cố lớn |
| `group_by` | `[alertname, host]` | Gom theo loại + máy |
| `send_resolved` | `true` cho critical, `false` cho info | Báo khi hết alert |
| `amtool silence` | Thủ công khi maintenance | Tắt tạm thời |

---

## 4. "Valerter" — clarification

Tên **"Valerter"** không phải tool chính thức của VictoriaMetrics. 3 khả năng:

| Khả năng | Nhận diện | Hành động |
|---|---|---|
| Typo của **vmalert** | Trong Notion viết "Valerter" để chỉ vmalert | Đang dùng rồi — chính là stack hiện tại |
| **Custom tool** (community) | Repo riêng từ Notion bạn đọc | Cần link cụ thể để xem |
| **Grafana Alerting** | Alert engine riêng của Grafana, bypass AM | Đổi sang nếu muốn UI quản lý mạnh |

**Brutal honesty**: stack hiện tại (vmalert + Alertmanager + Telegram) là **chuẩn industry**, đầy đủ tính năng (group/inhibit/silence/multi-receiver). Không cần thay nếu không có lý do cụ thể.

Nếu muốn nâng UX:
- **Karma**: web UI quản lý alert/silence dễ hơn AM UI mặc định.
- **Grafana Alerting**: tích hợp dashboard + alert + contact point Telegram trực tiếp (không cần AM).

---

## 5. Quy trình rollout

1. **Backup config cũ:**
   ```bash
   sudo cp /etc/vmalert/rules/logs.yml /etc/vmalert/rules/logs.yml.bak
   sudo cp /etc/alertmanager/alertmanager.yml /etc/alertmanager/alertmanager.yml.bak
   ```

2. **Apply rules mới** (mục 1) + **AM config mới** (mục 2):
   ```bash
   sudo nano /etc/vmalert/rules/logs.yml
   sudo nano /etc/alertmanager/alertmanager.yml
   amtool check-config /etc/alertmanager/alertmanager.yml
   sudo systemctl restart vmalert alertmanager
   ```

3. **Test từng alert:**
   ```bash
   # SSH brute force
   for i in $(seq 1 10); do logger -t sshd "Failed password test"; done

   # OOM giả lập
   logger -t kernel -p kern.crit "Out of memory: Killed process 1234 (test)"

   # Kernel panic giả lập
   logger -t kernel -p kern.emerg "BUG: kernel NULL pointer dereference"

   # Disk full
   logger -t kernel -p kern.err "No space left on device"

   # Root login
   logger -t sshd "Accepted password for root from 1.2.3.4"

   # Sudo abuse
   for i in $(seq 1 5); do logger -t sudo "pam_unix(sudo:auth): authentication failure"; done
   ```

4. **Observ 1 tuần** → điều chỉnh threshold + silence pattern không hữu ích.

5. **Tài liệu hóa**: ghi vào `docs/alerting-runbook.md` từng alert → ý nghĩa → action khi nhận.

---

## 6. Anti-pattern cần tránh

| Sai | Hậu quả | Đúng |
|---|---|---|
| `expr: count > 0` cho mọi log | Spam liên tục | Đặt ngưỡng + window phù hợp |
| `interval: 1s` | Vmalert ngốn CPU, query VL liên tục | Tối thiểu 5-10s |
| `repeat_interval: 5s` | Telegram spam, có thể bị rate limit | 15m+ cho critical, vài giờ cho info |
| Tất cả severity vào 1 chat | Mất context, info che critical | Tách 3 group: critical / security / info |
| Không có `for:` | Alert flapping (fire/resolve liên tục) | Critical 0s, warning 30s+ |
| Hardcode threshold không observ | Hoặc miss alert, hoặc spam | Đo baseline 1 tuần, set p95 + 20% |
| Không có inhibit | Khi host chết → ngập warning cùng host | Inhibit critical mute warning cùng host |

---

## 7. Roadmap nâng cấp (tùy chọn)

| Giai đoạn | Hành động |
|---|---|
| **Now** | Apply mục 1-2, observ |
| **Tuần 1** | Tinh chỉnh ngưỡng theo baseline |
| **Tuần 2** | Thêm rule riêng cho từng app (nginx 5xx, mysql slowlog...) |
| **Tháng 1** | Cài Karma UI để quản lý silence dễ hơn |
| **Tháng 2** | Thêm runbook link vào annotation: `runbook_url: "wiki/runbook/SSHBruteForce"` |
| **Tháng 3** | Auto-remediation: webhook AM → script tự ban IP brute-force |

---

## Unresolved

- "Valerter" trong Notion là tool nào cụ thể? Cần link để xác định có cần thay vmalert không.
- Baseline log volume thực tế của client-01? (cần observ 1 tuần để tinh threshold).
- Maintenance window có lịch cố định không? (để dùng `amtool silence` recurring).
