# Các hệ thống Log Server tập trung

## Tổng quan kiến trúc chung

```
[Sources]          [Collector/Agent]      [Pipeline/ETL]     [Storage + Search]   [Visualization]
Linux/Win/App  →   rsyslog / Fluentd /  → Logstash /     →   ES / Loki /      →  Kibana / Grafana /
Network/FW          Filebeat / Vector       Vector            VictoriaLogs        Splunk UI
                    NXLog                                     Clickhouse
```

5 thành phần lõi: **Thu thập → Vận chuyển → Xử lý/Chuẩn hóa → Lưu trữ + Index → Truy vấn/Trực quan hóa**.

---

## 1. ELK / Elastic Stack (Elasticsearch + Logstash + Kibana)

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Elasticsearch (Apache Lucene, inverted index) |
| **Pipeline** | Logstash (Ruby, nặng) hoặc Beats (Filebeat, nhẹ) |
| **UI** | Kibana — dashboard mạnh, query DSL/KQL |
| **Mã nguồn** | SSPL (không còn fully OSS từ 2021) |
| **Ưu** | Hệ sinh thái rộng, search cực mạnh, alert/ML có sẵn |
| **Nhược** | Ngốn RAM/disk (đặc biệt ES), license phức tạp |
| **Dùng khi** | Cần full-text search mạnh, dashboard đa dạng, có team chuyên |

> **Biến thể OSS thay thế:** **OpenSearch** (AWS fork từ ES 7.10).

---

## 2. Grafana Loki

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Object storage (S3, GCS) + chunk index nhẹ |
| **Pipeline** | Promtail / Fluent Bit / Vector / Grafana Agent |
| **UI** | Grafana — LogQL (giống PromQL) |
| **Mã nguồn** | AGPLv3 (Grafana Labs) |
| **Ưu** | **Rẻ** (chỉ index label), tích hợp Prometheus/Grafana tự nhiên |
| **Nhược** | Full-text search chậm hơn ES; không hợp khi grep sâu trên TB log |
| **Dùng khi** | Đã có Grafana, môi trường K8s, query theo label |

---

## 3. VictoriaLogs

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Định dạng nén riêng, ít CPU/RAM nhất trong nhóm |
| **Pipeline** | Vector / Fluent Bit / rsyslog (HTTP) |
| **UI** | Grafana plugin / web UI riêng / LogsQL |
| **Mã nguồn** | Apache 2.0 (VictoriaMetrics team) |
| **Ưu** | **Nhẹ nhất** (1 binary Go), nhanh, ingest rate cao |
| **Nhược** | Sinh thái còn non, ít plugin alerting |
| **Dùng khi** | Hạ tầng nhỏ–vừa, ưu tiên hiệu năng & chi phí |

> Xem chi tiết riêng: [`victorialogs.md`](./victorialogs.md)

---

## 4. Splunk

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Index proprietary |
| **Pipeline** | Universal Forwarder / Heavy Forwarder / HEC API |
| **UI** | Splunk Web — SPL cực mạnh |
| **Mã nguồn** | **Thương mại** (license theo GB/ngày, đắt) |
| **Ưu** | Tốt nhất market về search/analytic, SIEM (Splunk ES, SOAR) |
| **Nhược** | **Rất đắt**, vendor lock-in |
| **Dùng khi** | Doanh nghiệp lớn, SOC, compliance |

---

## 5. Graylog

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Elasticsearch/OpenSearch + MongoDB (metadata) |
| **Pipeline** | Inputs native (Syslog, GELF, Beats, Kafka...) |
| **UI** | Graylog Web — Streams, Pipelines, Dashboards |
| **Mã nguồn** | SSPL + bản Enterprise |
| **Ưu** | Cài đặt nhanh, UI thân thiện admin, RBAC/audit tốt |
| **Nhược** | Vẫn phụ thuộc ES → tốn tài nguyên |
| **Dùng khi** | Cần ELK-like nhưng đỡ phức tạp |

---

## 6. ClickHouse + UI (Signoz, Metabase, custom)

| Đặc điểm | Chi tiết |
|---|---|
| **Storage** | Columnar OLAP DB |
| **Pipeline** | Vector / Fluent Bit / Kafka |
| **UI** | Signoz, Metabase, Grafana plugin |
| **Mã nguồn** | Apache 2.0 |
| **Ưu** | Query analytics cực nhanh, nén tốt, scale ngang dễ |
| **Nhược** | Không phải full-text search engine; cần thiết kế schema |
| **Dùng khi** | Cần aggregate/group by, observability với Signoz |

---

## 7. Cloud-managed (SaaS)

| Tên | Đặc trưng |
|---|---|
| **Datadog Logs** | All-in-one observability, đắt, UX cực tốt |
| **AWS CloudWatch Logs** | Native AWS, integrate IAM/Lambda |
| **GCP Cloud Logging** | Native GCP, có Log Router |
| **Azure Monitor (Log Analytics)** | KQL mạnh, native Azure |
| **Logz.io / Better Stack / Logtail** | ELK/Loki SaaS |
| **Sumo Logic / New Relic Logs** | Enterprise observability |

→ **Ưu:** Không phải vận hành. **Nhược:** Chi phí egress + lock-in.

---

## 8. Kafka — "**buffer**" giữa các tầng

Không phải log server nhưng quan trọng trong pipeline production:
- Buffer khi storage chết / quá tải
- Fan-out 1 luồng log đến nhiều consumer
- Replay/reprocess khi parser sai

```
[Sources] → [Agent] → [KAFKA] → [Consumer 1: ES]
                              → [Consumer 2: VictoriaLogs]
                              → [Consumer 3: Cold storage S3]
                              → [Consumer 4: SIEM]
```

---

## So sánh nhanh

| Tiêu chí | ELK | Loki | VictoriaLogs | Splunk | Graylog | ClickHouse |
|---|---|---|---|---|---|---|
| **Full-text search** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Analytics/agg** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Resource cost** | Cao | Thấp | Rất thấp | TB | Cao | Thấp |
| **License** | SSPL | AGPL | Apache 2 | Commercial | SSPL | Apache 2 |
| **Learning curve** | Cao | TB | Thấp | TB | Thấp | Cao |
| **Phù hợp scale** | Lớn | Rất lớn | Vừa-Lớn | Rất lớn | Vừa | Rất lớn |

---

## Chọn theo nhu cầu

| Tình huống | Lựa chọn |
|---|---|
| Mới bắt đầu, lab/homelab | **Graylog** hoặc **Loki + Grafana** |
| Startup/SME, ưu tiên rẻ | **VictoriaLogs** hoặc **Loki** |
| Cần search mạnh, dashboard đẹp | **OpenSearch + Kibana** (free) |
| Enterprise, có ngân sách | **Splunk** hoặc **Datadog** |
| Đã dùng Prometheus/Grafana | **Loki** (native) |
| Cần analytics OLAP nặng | **ClickHouse + Signoz** |
| K8s-native, observability | **Signoz / Loki / OpenTelemetry stack** |
| Production multi-sink, chống mất log | **Vector + Kafka + (ES/Loki/VL)** |

---

## Kiến trúc production tham chiếu

```
                                 ┌────────────────┐
[Linux/Win/Net/FW] ──► rsyslog ──┤                ├──► VictoriaLogs (hot, search nhanh)
                                 │   VECTOR       │
[K8s/App pods]    ──► Fluent ────┤  (parse,route) ├──► ELK / OpenSearch (analytics, SIEM)
                                 │                │
                                 │                ├──► Kafka ──► Cold S3 (compliance)
                                 └────────────────┘
                                        │
                                        └──► Alertmanager / Grafana / Splunk ES
```

**Pattern phổ biến:**
- **rsyslog/Fluent Bit** ở edge (nhẹ, ổn định)
- **Vector** ở aggregator (parse mạnh, fan-out)
- **Kafka** làm buffer
- **Storage** chia hot/warm/cold theo chi phí
- **Grafana** unified dashboard (logs + metrics + traces)

---

## Câu hỏi cần làm rõ trước khi chọn

1. **Volume**: bao nhiêu GB/ngày?
2. **Retention**: lưu bao lâu? (compliance?)
3. **Use case**: debug, analytics, SIEM, audit?
4. **Team skill**: có người vận hành ES/Kafka không?
5. **Budget**: self-host hay SaaS?
6. **Integration**: đã có Prometheus/Grafana? AWS/GCP?
