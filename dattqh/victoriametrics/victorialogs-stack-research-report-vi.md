# Tìm hiểu VictoriaLogs: thu thập và truy vấn log tập trung

## 1. Mục tiêu

Báo cáo này dựng một hệ thống log tập trung tối giản bằng VictoriaLogs trên một máy Ubuntu, cho rsyslog đẩy log hệ thống vào, rồi truy vấn lại bằng LogsQL. Mục tiêu cụ thể: cài được VictoriaLogs single-node chạy dưới systemd, đưa log thật qua đường syslog, và đọc được output của ít nhất ba kiểu truy vấn LogsQL (lọc theo từ khoá, lọc theo trường, và thống kê).

VictoriaMetrics chỉ được nhắc tới như phần còn lại của cùng hệ sinh thái, không phải trọng tâm.

## 2. VictoriaLogs là gì, đứng ở đâu

VictoriaLogs là cơ sở dữ liệu log mã nguồn mở của nhóm VictoriaMetrics. Từ giữa năm 2025 nó được tách thành kho mã riêng `github.com/VictoriaMetrics/VictoriaLogs`, không còn nằm chung với VictoriaMetrics core. Bản dùng trong báo cáo là `v1.50.0`.

Điểm khác với Elasticsearch — vốn hay được mặc định khi nói tới "lưu log" — nằm ở hai chỗ. Thứ nhất, dữ liệu được lưu theo cột và nén theo từng luồng log thay vì theo mô hình tài liệu của Lucene. Thứ hai, truy vấn dùng ngôn ngữ riêng LogsQL chứ không phải Query DSL; khả năng tương thích Elasticsearch chỉ có ở khâu nhận log (API `_bulk`), theo tài liệu [data ingestion: Elasticsearch bulk API](https://docs.victoriametrics.com/victorialogs/data-ingestion/#elasticsearch-bulk-api). Vì vậy không cắm Kibana vào VictoriaLogs; phần xem log dùng giao diện tích hợp `vmui` hoặc datasource VictoriaLogs trong Grafana.

Quan hệ với VictoriaMetrics: hai sản phẩm cùng nhóm, cùng triết lý vận hành nhẹ, nhưng tách biệt. VictoriaMetrics + MetricsQL lo metrics; VictoriaLogs + LogsQL lo logs.

## 3. Mô hình dữ liệu

Mỗi dòng log trong VictoriaLogs có ba trường đặc biệt, mô tả trong [key concepts](https://docs.victoriametrics.com/victorialogs/keyconcepts/):

- `_msg` — nội dung thông điệp.
- `_time` — mốc thời gian của log.
- `_stream` (và `_stream_id`) — định danh luồng, là tập hợp các "stream fields" được chọn lúc ingest.

Các trường còn lại là trường thường. Một luồng (`_stream`) gom các log có cùng bộ stream fields, gần giống khái niệm label của Loki. Chọn stream fields là quyết định quan trọng: nên dùng trường ổn định, ít giá trị như `app_name`, `hostname`; nếu lỡ đưa trường biến thiên cao (ví dụ `request_id`) vào stream fields thì số luồng phình lên gần bằng số dòng log, làm tốn RAM và chậm ingest. Đây là vấn đề "high cardinality" quen thuộc với người từng dùng Prometheus hay Loki.

## 4. Cài đặt thực tế

Môi trường: Ubuntu 24.04 LTS, kernel `6.8.0-31-generic`, kiến trúc x86_64; rsyslog có sẵn và đang chạy.

Tải binary single-node từ kho VictoriaLogs và kiểm tra phiên bản:

```
$ ./victoria-logs-prod --version
victoria-logs-20260414-184100-tags-v1.50.0-0-g80d223f95f
```

Đặt binary vào `/usr/local/bin/victoria-logs` rồi tạo một service systemd. Hai cờ quan trọng: `-httpListenAddr=:9428` mở cổng HTTP cho truy vấn và UI, `-syslog.listenAddr.tcp=:5410` bật nhận log syslog (theo [data ingestion: syslog](https://docs.victoriametrics.com/victorialogs/data-ingestion/syslog/)):

```
[Service]
ExecStart=/usr/local/bin/victoria-logs \
  -storageDataPath=/var/lib/victoria-logs \
  -httpListenAddr=:9428 \
  -syslog.listenAddr.tcp=:5410
Restart=always
```

Sau khi `systemctl enable --now`, service ở trạng thái active và hai cổng cùng mở:

```
$ systemctl is-active victorialogs
active
$ ss -ltn | grep -E ':9428|:5410'
LISTEN 0  4096  0.0.0.0:9428  0.0.0.0:*
LISTEN 0  4096  0.0.0.0:5410  0.0.0.0:*
```

Một binary phục vụ cả ingest, lưu trữ và query — đó là đặc trưng của bản single-node.

## 5. Đưa log vào bằng rsyslog

Thay vì cài thêm collector, báo cáo dùng luôn rsyslog đang chạy. Một file cấu hình nhỏ trong `/etc/rsyslog.d/` cho rsyslog forward toàn bộ log local sang cổng 5410 theo định dạng RFC 5424:

```
*.* action(type="omfwd" target="127.0.0.1" port="5410" protocol="tcp" template="RSYSLOG_SyslogProtocol23Format")
```

Template `RSYSLOG_SyslogProtocol23Format` chính là khung RFC 5424 mà VictoriaLogs hiểu được. Sau khi restart rsyslog, sinh vài dòng log thử:

```
$ logger -t demo "hello victorialogs from rsyslog"
$ logger -t demo -p user.err "simulated error connection refused to db"
```

VictoriaLogs parse RFC 5424 và tự tách trường. Một dòng log nhìn thấy qua truy vấn có dạng:

```
{"_msg":" hello victorialogs from rsyslog",
 "_stream":"{app_name=\"demo\",hostname=\"dattqh-nat\",proc_id=\"-\"}",
 "_time":"2026-06-11T08:02:40.975706Z",
 "app_name":"demo","hostname":"dattqh-nat","level":"notice",
 "severity":"5","facility":"1","format":"rfc5424"}
```

Đáng chú ý: dòng `logger -p user.err` được ánh xạ thành `level=error` (severity 3), còn dòng thường thành `level=notice`. Trường `app_name`, `hostname` được đặt vào `_stream`. Đây là các tên trường thật sẽ dùng khi viết LogsQL.

## 6. Truy vấn bằng LogsQL

Endpoint truy vấn là `/select/logsql/query`, nhận tham số `query` và trả về JSON mỗi dòng một log (theo [querying](https://docs.victoriametrics.com/victorialogs/querying/)). Ba kiểu lọc cần phân biệt rõ:

**Word filter** — từ khoá trần khớp chuỗi con trong `_msg`:

```
$ curl -s -G 'http://localhost:9428/select/logsql/query' --data-urlencode 'query=error'
{"_msg":" simulated error connection refused to db", ... "level":"error", ...}
```

Lưu ý một cái bẫy: truy vấn `demo` không trả gì, vì "demo" nằm ở `app_name` chứ không nằm trong `_msg`. Word filter chỉ soi `_msg`.

**Stream filter** — lọc theo nguồn, dùng đúng cho trường hợp trên:

```
$ curl -s -G 'http://localhost:9428/select/logsql/query' --data-urlencode 'query=_stream:{app_name="demo"}'
```

Truy vấn này trả về cả hai dòng log của `app_name=demo`.

**Thống kê bằng pipe** — LogsQL nối các bước qua dấu `|`. Đếm log theo mức độ:

```
$ curl -s -G '.../select/logsql/query' --data-urlencode 'query=* | stats by (level) count() | sort by (count) desc'
{"level":"info","count(*)":"32"}
{"level":"notice","count(*)":"4"}
{"level":"warning","count(*)":"1"}
{"level":"error","count(*)":"1"}
```

Đếm theo ứng dụng cho thấy nguồn log nào ồn nhất:

```
$ curl -s -G '.../select/logsql/query' --data-urlencode 'query=* | stats by (app_name) count() | sort by (app_name)'
{"app_name":"sshd","count(*)":"8"}
{"app_name":"systemd","count(*)":"10"}
{"app_name":"systemd-logind","count(*)":"8"}
{"app_name":"rsyslogd","count(*)":"5"}
{"app_name":"demo","count(*)":"2"}
...
```

Kết hợp lọc thời gian và lọc trường để soi lỗi gần đây:

```
$ curl -s -G '.../select/logsql/query' --data-urlencode 'query=_time:5m level:error'
{"_msg":" simulated error connection refused to db", ... "level":"error", ...}
```

So với SQL hay PromQL, LogsQL đọc khác hẳn: bắt đầu bằng biểu thức lọc, rồi đẩy qua các pipe `stats`, `sort`, `limit`. Cú pháp đầy đủ ở [LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/).

Ngoài curl, giao diện `vmui` tại `http://<host>:9428/select/vmui` cho phép gõ cùng các truy vấn này và xem kết quả trực quan, lọc theo thời gian.

## 7. Kiến trúc single-node và cluster

Bản single-node dùng trong báo cáo là một tiến trình làm tất cả. Khi một node không còn đủ cho lượng log, VictoriaLogs có bản cluster tách ba vai trò, mô tả ở [cluster](https://docs.victoriametrics.com/victorialogs/cluster/):

- `vlinsert` — nhận log và định tuyến tới nơi lưu.
- `vlstorage` — lưu trữ log nén theo cột.
- `vlselect` — nhận truy vấn LogsQL, gom kết quả từ các `vlstorage`.

Cách chia này song song với bản cluster của VictoriaMetrics (vminsert/vmstorage/vmselect), nên ai quen một bên sẽ thấy bên kia quen thuộc. Khuyến nghị hợp lý là khởi đầu bằng single-node và chỉ chuyển cluster khi thực sự cần scale, vì cluster thêm chi phí vận hành.

## 8. Nhận xét

Điểm mạnh thấy rõ qua lần dựng này là chi phí khởi động thấp: một binary, một file systemd, một dòng cấu hình rsyslog là đã có log tập trung truy vấn được. Việc tận dụng rsyslog có sẵn giúp không phải thêm tác nhân thu log nào trên máy.

Điểm cần nắm để khỏi mất thời gian: phân biệt word filter (soi `_msg`) với stream filter (`_stream:{...}`). Lần đầu truy vấn `demo` ra rỗng dễ làm tưởng log chưa vào, trong khi thực ra log đã vào nhưng từ khoá nằm ở trường khác. Một điểm thiết kế cần cân nhắc khi triển khai thật là chọn stream fields: giữ ở mức `app_name`/`hostname` để tránh high cardinality.

Hướng tìm hiểu tiếp: thử các đường ingest khác (Vector, OpenTelemetry), cấu hình thời gian lưu giữ (retention), và gắn datasource VictoriaLogs vào Grafana để dựng dashboard.

## 9. Nguồn tham khảo

- [1] VictoriaLogs — tổng quan. https://docs.victoriametrics.com/victorialogs/
- [2] Key concepts (fields, streams). https://docs.victoriametrics.com/victorialogs/keyconcepts/
- [3] LogsQL. https://docs.victoriametrics.com/victorialogs/logsql/
- [4] Data ingestion: syslog. https://docs.victoriametrics.com/victorialogs/data-ingestion/syslog/
- [5] Data ingestion: Elasticsearch bulk API. https://docs.victoriametrics.com/victorialogs/data-ingestion/#elasticsearch-bulk-api
- [6] Querying. https://docs.victoriametrics.com/victorialogs/querying/
- [7] Cluster. https://docs.victoriametrics.com/victorialogs/cluster/
- [8] Quickstart. https://docs.victoriametrics.com/victorialogs/quickstart/
- [9] Kho mã VictoriaLogs (v1.50.0). https://github.com/VictoriaMetrics/VictoriaLogs/releases
