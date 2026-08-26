# Tìm hiểu rsyslog: định tuyến và chuyển tiếp log hệ thống

## 1. Mục tiêu

Báo cáo này nhìn vào rsyslog — tác nhân syslog có sẵn trên Ubuntu — ở vai trò đầu nguồn của một đường log tập trung: gom log cục bộ, gắn nhãn theo facility/severity, rồi chuyển tiếp sang collector qua TCP theo định dạng RFC 5424. Đây là nửa "phát" của cùng đường ống đã nạp log vào VictoriaLogs trên máy `dattqh-nat` (Ubuntu 24.04); báo cáo VictoriaLogs lo nửa "nhận", báo cáo này lo nửa "phát". Mục tiêu cụ thể: hiểu pipeline xử lý của rsyslog, đọc được con số PRI trên log thật, viết được một quy tắc forward bằng `omfwd`, và biết khi nào cần queue/TLS để khỏi mất log.

## 2. rsyslog là gì, đứng ở đâu

rsyslog là bản kế thừa của sysklogd, mặc định trên hầu hết bản phân phối Linux (gồm Ubuntu 24.04 dùng ở đây), nên thường không phải cài thêm. Ngoài việc ghi log ra file như syslogd cổ điển, rsyslog còn là một khung xử lý và định tuyến sự kiện: nhận log từ nhiều nguồn, lọc, biến đổi, rồi đẩy đi nhiều đích.

Về định dạng, rsyslog nói được cả RFC 3164 (BSD syslog cũ — timestamp thô, không timezone) lẫn RFC 5424 (chuẩn mới — timestamp ISO 8601 có timezone, có structured-data, app-name, procid, msgid). Khi forward sang collector hiện đại nên chọn RFC 5424 vì tách trường sạch hơn.

So với hàng xóm: `syslog-ng` cùng phân khúc nhưng khác cú pháp; `systemd-journald` là log nhị phân cục bộ của systemd. Thường journald và rsyslog chạy song song — journald giữ log nhị phân cho `journalctl`, còn rsyslog lo phần ghi file, định tuyến và forward đi xa.

## 3. Mô hình xử lý: input → ruleset → output

Theo [basic structure](https://docs.rsyslog.com/doc/configuration/basic_structure.html), log chạy qua ba chặng — module input → ruleset (các rule = filter cộng danh sách action) → module output — kèm parser, template ở giữa và queue đệm giữa các chặng.

- Input module, tiền tố `im*`: `imuxsock` (socket `/dev/log` cục bộ), `imjournal` (lấy từ journald), `imudp`/`imtcp` (nhận syslog qua mạng), `imfile` (theo dõi file text).
- Output module (action), tiền tố `om*`: `omfile` (ghi file), `omfwd` (forward syslog), `omelasticsearch`, `ommysql`…
- Ruleset: tập rule chạy tuần tự từ trên xuống; mỗi rule gồm một filter và các action; `stop` để dừng xử lý dòng log đó.
- Template định khuôn chuỗi xuất; parser tách RFC 3164/RFC 5424/JSON.

Hai cú pháp cấu hình cùng tồn tại: kiểu sysklogd cũ (dòng selector `facility.severity  action`) và RainerScript hiện đại (khối `action(...)`, `if ... then`). RainerScript là format được khuyến nghị; selector cũ vẫn chạy, tiện cho cấu hình một dòng.

## 4. Facility, severity và số PRI

Mỗi message syslog mang hai nhãn: facility (nguồn sinh log — `kern`, `user`, `mail`, `daemon`, `auth`…) và severity (mức — `emerg`=0 … `debug`=7). Hai nhãn này gói thành một số PRI duy nhất:

```
PRI = facility * 8 + severity
```

Số PRI nằm trong cặp `<>` đầu mỗi dòng RFC 5424. Đối chiếu trên log thật mà rsyslog đẩy vào VictoriaLogs ở lab:

| Nguồn | facility | severity | PRI | level |
|---|---|---|---|---|
| `logger "hello…"` | user (1) | notice (5) | 13 | notice |
| `logger -p user.err …` | user (1) | err (3) | 11 | error |
| systemd-logind | auth (4) | info (6) | 38 | info |
| systemd | daemon (3) | info (6) | 30 | info |

Một dòng thật, nguyên văn từ collector:

```
{"_msg":" hello victorialogs from rsyslog",
 "_stream":"{app_name=\"demo\",hostname=\"dattqh-nat\",proc_id=\"-\"}",
 "_time":"2026-06-11T08:02:40.975706Z",
 "app_name":"demo","facility":"1","facility_keyword":"user",
 "format":"rfc5424","level":"notice","priority":"13","severity":"5"}
```

`priority:13` đúng bằng `1*8 + 5` — facility `user`, severity `notice`. Đọc được PRI giúp hiểu nhanh log thô và viết selector lọc đúng mức.

## 5. Cấu hình thực tế

rsyslog chạy sẵn nên không có bước cài. Cấu hình ở `/etc/rsyslog.conf` (file chính) và các drop-in `/etc/rsyslog.d/*.conf` nạp theo thứ tự tên file. Quy ước là thêm một file riêng trong `rsyslog.d/` thay vì sửa file chính, để nâng cấp gói không đụng vào tuỳ biến.

Kiểm tra service đang chạy:

```
$ systemctl is-active rsyslog
active
```

Validate cú pháp trước khi restart, tránh để daemon chết vì config sai:

```
$ rsyslogd -N1
```

Áp dụng thay đổi bằng `systemctl restart rsyslog`.

## 6. Định tuyến và lọc log

Theo [filter conditions](https://docs.rsyslog.com/doc/configuration/filters.html), rsyslog lọc theo ba kiểu chính.

Selector truyền thống `facility.severity` — ngắn gọn khi lọc theo mức:

```
auth,authpriv.*    /var/log/auth.log
mail.err           /var/log/mail.err
*.*                @@127.0.0.1:5410
```

Trong cú pháp cũ: `.severity` nghĩa là "mức này trở lên"; `*` là mọi facility hoặc mọi mức; `@@` là TCP, `@` là UDP.

Property-based filter — lọc theo nội dung trường, ví dụ theo tên chương trình:

```
:programname, isequal, "sshd"    /var/log/sshd.log
```

Expression-based (RainerScript) — điều kiện đầy đủ, ghép `and`/`or`, kèm `stop`:

```
if $programname == "demo" and $syslogseverity <= 3 then {
    action(type="omfwd" target="127.0.0.1" port="5410" protocol="tcp"
           template="RSYSLOG_SyslogProtocol23Format")
    stop
}
```

## 7. Chuyển tiếp sang collector bằng omfwd

`omfwd` là module output forward log qua UDP/TCP/TLS, dựng sẵn trong rsyslog nên không cần `module(load=...)` (theo [omfwd](https://docs.rsyslog.com/doc/configuration/modules/omfwd.html)). Toàn bộ việc nạp log cho VictoriaLogs ở lab chỉ là một dòng trong `/etc/rsyslog.d/`:

```
*.* action(type="omfwd" target="127.0.0.1" port="5410" protocol="tcp" template="RSYSLOG_SyslogProtocol23Format")
```

Đọc dòng này: mọi log (`*.*`) forward qua TCP tới `127.0.0.1:5410` (cổng syslog của VictoriaLogs), đóng khung theo template `RSYSLOG_SyslogProtocol23Format`.

Template đó là một template reserved có sẵn, sinh đúng khung RFC 5424 ([reserved template names](https://docs.rsyslog.com/doc/reference/templates/templates-reserved-names.html)):

```
<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n
```

Chính khung này giải thích vì sao collector tách được trường: `%PRI%` → `priority`/`facility`/`severity`, `%HOSTNAME%` → `hostname`, `%APP-NAME%` → `app_name`, `%PROCID%` → `proc_id`, `%MSGID%` → `msg_id`, `%msg%` → `_msg` (giữ một khoảng trắng đầu, thấy rõ ở `" hello…"`). Chọn sai template — ví dụ khung file truyền thống — thì collector chỉ nhận một chuỗi thô, mất hết trường.

TCP (`@@`) so với UDP (`@`): UDP nhẹ nhưng mất gói là mất log không báo; TCP có bắt tay, hợp với log cần đủ. Lab dùng TCP.

## 8. Độ tin cậy: queue và mã hoá

`omfwd` gắn với hệ thống queue của rsyslog ([queues](https://docs.rsyslog.com/doc/concepts/queues.html)): action chạy qua một hàng đợi, mặc định in-memory. Khi collector tạm chết, log mới kẹt lại và sẽ mất nếu chỉ có queue bộ nhớ rồi rsyslog restart. Disk-assisted queue ghi tràn ra đĩa, giữ log qua lúc collector gián đoạn rồi gửi bù khi nối lại:

```
action(type="omfwd" target="127.0.0.1" port="5410" protocol="tcp"
       template="RSYSLOG_SyslogProtocol23Format"
       queue.type="linkedlist" queue.filename="fwd_vl"
       queue.maxDiskSpace="1g" action.resumeRetryCount="-1")
```

Khi log đi qua mạng không tin cậy: `omfwd` hỗ trợ TLS (qua driver `gtls`); cần giao nhận chắc hơn nữa thì dùng `omrelp` (giao thức RELP có xác nhận đến nơi). Lab chạy nội bộ qua loopback nên để mặc định.

## 9. Nhận xét

Điểm mạnh: rsyslog có sẵn, và một dòng `omfwd` đủ biến mọi log cục bộ thành luồng chuyển tiếp — không thêm tác nhân thu log nào, đúng tinh thần tận dụng hạ tầng đã có.

Điểm cần nắm để khỏi mất thời gian: (1) chọn đúng template RFC 5424 — sai template thì collector mất trường, lỗi này im lặng vì log vẫn "vào" nhưng không query theo trường được; (2) phân biệt facility/severity và đọc được PRI để viết selector đúng mức; (3) mặc định queue bộ nhớ cộng UDP là tổ hợp dễ mất log — môi trường thật nên dùng TCP và bật disk-assisted queue.

Hướng tìm hiểu tiếp: nhận log từ máy khác qua `imtcp`/`imudp` (rsyslog làm server tập trung), bật TLS giữa client và server, và so `omfwd` (syslog thuần) với `omrelp`/`omelasticsearch` cho các collector khác.

## 10. Nguồn tham khảo

- [1] rsyslog — basic structure (pipeline). https://docs.rsyslog.com/doc/configuration/basic_structure.html
- [2] Filter conditions (selector, property, expression). https://docs.rsyslog.com/doc/configuration/filters.html
- [3] omfwd — output module forward. https://docs.rsyslog.com/doc/configuration/modules/omfwd.html
- [4] Templates. https://docs.rsyslog.com/doc/configuration/templates.html
- [5] Reserved template names (RSYSLOG_SyslogProtocol23Format ≈ RFC 5424). https://docs.rsyslog.com/doc/reference/templates/templates-reserved-names.html
- [6] Queues (in-memory vs disk-assisted). https://docs.rsyslog.com/doc/concepts/queues.html
- [7] RFC 5424 — The Syslog Protocol. https://www.rfc-editor.org/rfc/rfc5424
- [8] Báo cáo liên quan: Tìm hiểu VictoriaLogs (nửa nhận của cùng đường log). ./victorialogs-stack-research-report-vi.md
