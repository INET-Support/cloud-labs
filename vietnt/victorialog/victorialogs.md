# VictoriaLogs

> Hệ thống quản lý log tập trung mã nguồn mở, **nhẹ – nhanh – rẻ**, được phát triển bởi VictoriaMetrics team. Cùng họ với VictoriaMetrics (time-series DB).

---

## 1. VictoriaLogs là gì?

- **Log database** mã nguồn mở (Apache 2.0), viết bằng **Go**, chỉ 1 binary duy nhất.
- Thiết kế để giải quyết các vấn đề của ELK/Loki:
  - **Ngốn tài nguyên** (ES)
  - **Full-text search yếu** (Loki)
  - **Khó vận hành** (cluster ES/OpenSearch)
- Mục tiêu: **ingest cao, query nhanh, chi phí thấp**.

---

## 2. Đặc điểm nổi bật

| Tính năng | Mô tả |
|---|---|
| **1 binary Go** | Không phụ thuộc JVM/Ruby, deploy chỉ 1 file |
| **Resource thấp** | ~30× ít RAM hơn ES, ~10× ít disk hơn Loki |
| **Ingest rate cao** | Lên tới hàng triệu log/s trên 1 node |
| **LogsQL** | Query language riêng, đơn giản hơn ES DSL, mạnh hơn LogQL |
| **Full-text search** | Index full-text mặc định (khác Loki chỉ index label) |
| **Multi-tenant** | Hỗ trợ nhiều tenant trên cùng cluster |
| **Schema-less** | Tự nhận field từ JSON log |
| **HTTP API** | Ingest qua HTTP JSON Lines, query qua HTTP |
| **Tích hợp Grafana** | Plugin chính thức `victorialogs-datasource` |

---

## 3. Kiến trúc

### Single-node (homelab / SME)
```
[rsyslog / Vector / Fluent Bit] → HTTP → [VictoriaLogs binary] → Local disk
                                              ↓
                                          [Grafana]
```

### Cluster (production scale)
```
[Agents] → [vlinsert]  (ingest, dedup, route)
              ↓
          [vlstorage]  (lưu data, nén)
              ↓
          [vlselect]   (query, aggregate)
              ↓
          [Grafana / API]
```

---

## 4. Cài đặt nhanh (single-node)

### 4.1. Tải binary
```bash
VL_VERSION="v1.7.0"   # check release mới nhất tại github.com/VictoriaMetrics/VictoriaLogs
wget https://github.com/VictoriaMetrics/VictoriaLogs/releases/download/${VL_VERSION}/victoria-logs-linux-amd64-${VL_VERSION}.tar.gz
tar xzf victoria-logs-linux-amd64-*.tar.gz
sudo mv victoria-logs-prod /usr/local/bin/victoria-logs
```

### 4.2. Tạo systemd service
```bash
sudo tee /etc/systemd/system/victorialogs.service > /dev/null <<'EOF'
[Unit]
Description=VictoriaLogs
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/victoria-logs \
  -storageDataPath=/var/lib/victorialogs \
  -httpListenAddr=:9428 \
  -retentionPeriod=30d
Restart=always
User=victorialogs

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -r -s /bin/false victorialogs
sudo mkdir -p /var/lib/victorialogs
sudo chown victorialogs: /var/lib/victorialogs
sudo systemctl daemon-reload
sudo systemctl enable --now victorialogs
```

### 4.3. Kiểm tra
```bash
curl http://localhost:9428/health
# → OK
```

### 4.4. Docker (nhanh hơn)
```bash
docker run -d --name victorialogs \
  -p 9428:9428 \
  -v /var/lib/victorialogs:/victoria-logs-data \
  victoriametrics/victoria-logs:latest \
  -retentionPeriod=30d
```

---

## 5. Ingest log

### 5.1. Từ rsyslog (HTTP JSON)
```conf
# /etc/rsyslog.d/95-victorialogs.conf
module(load="omhttp")

template(name="JsonLines" type="list" option.jsonf="on") {
    property(outname="_time"  name="timereported" dateFormat="rfc3339" format="jsonf")
    property(outname="host"   name="hostname"  format="jsonf")
    property(outname="app"    name="programname" format="jsonf")
    property(outname="severity" name="syslogseverity-text" format="jsonf")
    property(outname="_msg"   name="msg" format="jsonf")
}

action(
    type="omhttp"
    server="victorialogs-host"
    serverport="9428"
    restpath="insert/jsonline?_stream_fields=host,app&_time_field=_time&_msg_field=_msg"
    template="JsonLines"
    batch="on"
    batch.maxsize="1000"
)
```

> **Lưu ý field đặc biệt của VictoriaLogs:**
> - `_time` — timestamp (bắt buộc)
> - `_msg` — message chính (bắt buộc)
> - `_stream_fields` — các field định danh stream (host, app...) → giúp query nhanh

### 5.2. Từ Vector
```toml
[sinks.victorialogs]
type = "http"
inputs = ["normalize"]
uri = "http://victorialogs:9428/insert/jsonline?_stream_fields=host,app&_time_field=@timestamp&_msg_field=msg"
encoding.codec = "json"
framing.method = "newline_delimited"
batch.max_events = 1000
```

### 5.3. Từ Fluent Bit
```ini
[OUTPUT]
    Name           http
    Match          *
    Host           victorialogs
    Port           9428
    URI            /insert/jsonline?_stream_fields=host,app&_msg_field=log
    Format         json_lines
    json_date_key  _time
    json_date_format iso8601
```

### 5.4. Curl test
```bash
echo '{"_time":"2026-06-12T14:00:00Z","_msg":"hello victorialogs","host":"client-01","app":"test"}' \
  | curl -X POST -H 'Content-Type: application/json' \
    --data-binary @- 'http://localhost:9428/insert/jsonline?_stream_fields=host,app'
```

---

## 6. Query với LogsQL

### 6.1. Web UI
```
http://victorialogs:9428/select/vmui
```
→ UI có search box, time range picker, hit count chart.

### 6.2. HTTP API
```bash
curl 'http://localhost:9428/select/logsql/query' \
  --data-urlencode 'query=app:nginx error'
```

### 6.3. Cú pháp LogsQL cơ bản

| Mục đích | Query |
|---|---|
| Tìm chuỗi | `error` |
| Filter theo field | `app:nginx` |
| Kết hợp AND | `app:nginx status:500` |
| OR | `app:nginx OR app:apache` |
| NOT | `error NOT app:cron` |
| Regex | `_msg:~"timeout.*db"` |
| Time range | `_time:5m error` (5 phút gần nhất) |
| Count theo field | `error \| stats by (app) count()` |
| Top 10 host lỗi | `error \| stats by (host) count() \| sort by (count) desc \| limit 10` |

### 6.4. Ví dụ thực tế
```
# Tất cả log SSH thất bại trong 1h
app:sshd "Failed password" _time:1h

# Đếm log theo severity
_time:24h | stats by (severity) count()

# Top URI nginx trả 5xx
app:nginx status:~"5.." | stats by (uri) count() | sort by (count) desc | limit 20
```

---

## 7. Tích hợp Grafana

### 7.1. Cài plugin
```bash
grafana-cli plugins install victoriametrics-logs-datasource
sudo systemctl restart grafana-server
```

### 7.2. Thêm datasource
- **Type**: VictoriaLogs
- **URL**: `http://victorialogs:9428`
- Save & Test

### 7.3. Tạo dashboard
- Panel type: **Logs** hoặc **Time series**
- Query: cú pháp LogsQL như trên

---

## 8. Vận hành

### 8.1. Retention
```bash
-retentionPeriod=30d    # giữ 30 ngày
-retentionPeriod=1y     # giữ 1 năm
```

### 8.2. Disk usage
```bash
du -sh /var/lib/victorialogs/
# Compression ratio thường 10–50x so với raw log
```

### 8.3. Backup
```bash
# Snapshot (atomic, không khóa)
curl -X POST 'http://localhost:9428/snapshot/create'
# Trả về tên snapshot trong /var/lib/victorialogs/snapshots/
```

### 8.4. Metrics monitoring (Prometheus)
```bash
curl http://localhost:9428/metrics
# Expose Prometheus metrics → scrape bằng VictoriaMetrics/Prometheus
```

---

## 9. Ưu / Nhược điểm

### ✅ Ưu điểm
- Nhẹ nhất trong nhóm log DB (RAM, CPU, disk).
- Cài đặt cực nhanh (1 binary).
- LogsQL trực quan, mạnh.
- Ingest rate cao, scale ngang dễ.
- License Apache 2.0 — không lo lock-in.
- Full-text search **mặc định** (không phải đánh đổi như Loki).

### ❌ Nhược điểm
- Hệ sinh thái còn non so với ELK/Splunk.
- Ít built-in alerting (cần dùng kèm vmalert / Grafana alert).
- Cộng đồng/tài liệu nhỏ hơn ELK.
- Chưa có UI quản trị nhiều như Graylog/Kibana.
- Multi-tenant chưa mạnh bằng enterprise SIEM.

---

## 10. So với Loki / ELK

| Tiêu chí | VictoriaLogs | Loki | Elasticsearch |
|---|---|---|---|
| **Disk usage** | ⭐⭐⭐⭐⭐ (nén tốt nhất) | ⭐⭐⭐⭐ | ⭐⭐ |
| **RAM** | ~100MB–1GB | ~500MB–2GB | ~4GB+ |
| **Full-text** | ✅ Có | ⚠️ Yếu (chỉ label) | ✅ Mạnh nhất |
| **Setup** | 1 binary | Vừa | Phức tạp |
| **Query lang** | LogsQL | LogQL | ES DSL/KQL |
| **Aggregation** | Tốt | Yếu | Rất tốt |
| **Cluster** | Có | Có | Có |
| **Cộng đồng** | Nhỏ–TB | Lớn | Rất lớn |

---

## 11. Khi nào nên dùng VictoriaLogs?

✅ **Nên dùng khi:**
- Hạ tầng nhỏ–vừa, không có team chuyên ES
- Log volume lớn nhưng budget thấp
- Đã/đang dùng VictoriaMetrics (đồng bộ stack)
- Cần ingest rate cao + query nhanh
- Muốn deploy đơn giản, ít point-of-failure

❌ **Không nên dùng khi:**
- Cần SIEM enterprise (Splunk ES, QRadar)
- Cần ML/anomaly detection có sẵn (chọn ELK X-Pack hoặc Datadog)
- Đã có ELK chạy ổn, không có nhu cầu thay đổi
- Cần ecosystem plugin lớn (alert, ML, security app)

---

## 12. Tài liệu tham khảo

- Website: <https://docs.victoriametrics.com/victorialogs/>
- GitHub: <https://github.com/VictoriaMetrics/VictoriaLogs>
- LogsQL: <https://docs.victoriametrics.com/victorialogs/logsql/>
- Grafana plugin: <https://grafana.com/grafana/plugins/victoriametrics-logs-datasource/>
