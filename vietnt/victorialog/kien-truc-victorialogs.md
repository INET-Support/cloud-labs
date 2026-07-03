# Kiến trúc VictoriaLogs

> Tóm tắt kiến trúc VictoriaLogs (VL) — log database mã nguồn mở của VictoriaMetrics.
> Nguồn: https://victoriametrics.com/products/victorialogs/ • https://docs.victoriametrics.com/victorialogs/

---

## 1. Triết lý thiết kế

- **1 binary Go**, không JVM, không dependency runtime.
- Pipeline 3 giai đoạn: **Collect → Store → Analyze**.
- Tối ưu cho ingest rate cao, RAM/disk thấp (~30× ít RAM, ~15× ít disk so với Elasticsearch).
- Tỷ lệ nén ~50:1. 1 node VL ≈ thay thế cluster ES 30 node.

---

## 2. Mô hình triển khai

### 2.1. Single-node
Một binary `victoria-logs` đảm nhiệm cả ingest + storage + query. Mặc định cho mọi quy mô đến hàng TB/ngày.

```
        ┌────────────────────────────┐
agents  │  victoria-logs (1 binary)  │  ← HTTP/Syslog ingest + LogsQL query
  ───►  │  ingest │ storage │ query  │
        └────────────────────────────┘
                   │
                  disk (per-day partitions)
```

### 2.2. Cluster (phân tách 3 vai trò)

| Component | Vai trò |
|---|---|
| **vlinsert** | Stateless. Nhận log từ agent, băm theo `stream fields`, forward tới `vlstorage`. |
| **vlstorage** | Stateful. Lưu trữ trên disk, đánh index, nén dữ liệu. |
| **vlselect** | Stateless. Nhận query LogsQL, scatter-gather tới các `vlstorage`, gộp kết quả. |

```
   agents ──► vlinsert ──┐
                         ├──► vlstorage (N shards) ──┐
   Grafana ──► vlselect ─┘                            │
                         └──◄──── query fan-out ◄────┘
```

Scale ngang: tăng `vlinsert` cho ingest, `vlselect` cho query, `vlstorage` cho dung lượng.

---

## 3. Data Model

- **Schema-less**: log = tập key-value tùy ý, không cần khai báo trước.
- **Stream fields** (low-cardinality): `service`, `host`, `env`, `app` → dùng để gom log thành **log stream**, là đơn vị index gốc.
- **Log fields** (any-cardinality): mọi field còn lại, kể cả high-cardinality (`trace_id`, `user_id`, `request_id`).
- **`_time`**: timestamp bắt buộc.
- **`_msg`**: nội dung log gốc, được index full-text mặc định.
- **`_stream`**: định danh stream được sinh tự động từ stream fields.

> Khác Loki: VL index **toàn bộ** log field thay vì chỉ label → full-text search nhanh, không bị "needle in haystack".

---

## 4. Storage Engine

- **Per-day partitions**: mỗi ngày một thư mục độc lập. Xóa retention = `rm -rf` thư mục → cực nhanh.
- **Columnar storage**: mỗi field lưu riêng cột → query field nào chỉ đọc cột đó.
- **Compression**: ZSTD, dictionary encoding cho cột lặp.
- **Bloom filter + inverted index** cho từ khóa full-text.
- **Retention**:
  - Theo thời gian: `-retentionPeriod` (mặc định 7d).
  - Theo dung lượng: `-retention.maxDiskSpaceUsageBytes` → tự xóa partition cũ nhất khi quá ngưỡng.

---

## 5. Ingestion Protocols

| Protocol | Endpoint | Use case |
|---|---|---|
| **JSON stream** | `POST /insert/jsonline` | Native, khuyến nghị |
| **Elasticsearch bulk** | `POST /insert/elasticsearch/_bulk` | Drop-in cho Filebeat, Logstash, Fluentd |
| **Loki push** | `POST /insert/loki/api/v1/push` | Drop-in cho Promtail, Grafana Alloy |
| **Syslog (RFC 3164/5424)** | TCP/UDP 514 | rsyslog, syslog-ng, network device |
| **OpenTelemetry logs** | `POST /insert/opentelemetry/v1/logs` | OTel collector |
| **Datadog** | `POST /insert/datadog/api/v2/logs` | Datadog agent |
| **Journald** | qua vector/fluent-bit | systemd hosts |

Agent tương thích: **Fluent Bit, Vector, Filebeat, Logstash, Promtail, OpenTelemetry Collector, Telegraf, rsyslog, syslog-ng**.

---

## 6. Query Layer — LogsQL

- Ngôn ngữ pipe-style (giống Splunk SPL / Kusto), đơn giản hơn ES DSL, mạnh hơn LogQL.
- Pipeline operators: `filter`, `stats`, `sort`, `limit`, `extract`, `unpack_json`, `format`, `math`.
- Full-text + regex + range + stream filter trong cùng query.

Ví dụ:
```logsql
_time:5m service:"nginx" status:>=500
| extract "upstream=<upstream>"
| stats by (upstream) count() errors, quantile(0.99, response_time)
| sort by (errors desc)
```

### Query API
- `POST /select/logsql/query` — query thường.
- `GET /select/logsql/tail` — live tail (giống `tail -f`).
- `GET /select/logsql/stats_query` — phục vụ Grafana panel.

### UI/Client
- **Web UI** built-in tại `:9428/select/vmui/`.
- **vlogscli**: CLI tương tác có history, autocomplete, live tail.
- **Grafana**: plugin chính thức `victoriametrics-logs-datasource`.

---

## 7. Multi-tenancy

- Header `AccountID` + `ProjectID` trên request ingest & query → tách dữ liệu hoàn toàn ở storage layer.
- Không cần deploy nhiều instance cho nhiều team/khách hàng.

---

## 8. So sánh nhanh

| Tiêu chí | VictoriaLogs | Elasticsearch | Loki |
|---|---|---|---|
| Runtime | 1 binary Go | JVM | Go |
| RAM | Rất thấp | Cao | Trung bình |
| Disk | Rất thấp (50:1) | Cao | Thấp |
| Full-text index | Có, mặc định | Có | Không (chỉ label) |
| Query language | LogsQL (pipe) | ES DSL (JSON) | LogQL |
| High-cardinality | OK | Tốn RAM | Kém |
| Vận hành | Đơn giản | Phức tạp | Trung bình |

---

## 9. Khi nào dùng VL

✅ Centralized logging cho on-prem / hybrid cloud, ưu tiên TCO thấp.
✅ Thay thế ELK khi RAM/disk là nút thắt.
✅ Cần full-text search mà Loki không đáp ứng được.
✅ Tích hợp sẵn syslog → phù hợp ingest từ network device, Linux server.

⚠️ Hạn chế: hệ sinh thái alerting/visualization chưa phong phú bằng ELK; cluster mode còn mới (GA gần đây).

---

## 10. Tham khảo

- Trang sản phẩm: https://victoriametrics.com/products/victorialogs/
- Docs chính thức: https://docs.victoriametrics.com/victorialogs/
- LogsQL: https://docs.victoriametrics.com/victorialogs/logsql/
- Cluster: https://docs.victoriametrics.com/victorialogs/cluster/
