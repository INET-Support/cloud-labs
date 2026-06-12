# SYSLOG & RSYSLOG

> Ghi, truyền và quản lý log tập trung trong hệ thống Linux.

---

## 1. SYSLOG là gì?

- **Syslog** là một **chuẩn giao tiếp** dùng để ghi và truyền log giữa các hệ thống.
- Được ứng dụng hầu hết trên các hệ điều hành Linux/Unix.
- Mục tiêu: **gửi log tập trung**, dễ dàng **giám sát và phân tích**.

### Định dạng một Syslog Message
```
<PRI> TIMESTAMP HOSTNAME TAG: MESSAGE
```

| Thành phần | Ý nghĩa |
|------------|---------|
| **PRI**     | Mức độ ưu tiên = `Facility × 8 + Severity` |
| **TIMESTAMP** | Thời gian ghi log |
| **HOSTNAME**  | Tên máy gửi log |
| **TAG**       | Nguồn tạo log (vd: `sshd`, `cron`, ...) |
| **MESSAGE**   | Nội dung log |

---

## 2. RSYSLOG là gì?

- **rsyslog** là **daemon syslog** mạnh mẽ và mặc định trên đa số bản phân phối Linux hiện nay.
- Hỗ trợ ghi log cục bộ, gửi log qua mạng (TCP/UDP/TLS), chuyển hướng và lưu trữ nhiều định dạng.
- Ưu điểm: **hoạt động ổn định, hiệu năng cao, cấu hình linh hoạt**.

### Mô hình hoạt động
```
[Client / Log Source]  →  [RSYSLOG SERVER]  →  [Output / Destination]
   Web (nginx)            Input module          /var/log/ (local file)
   App (PHP)              Filter (lọc)          Database (MySQL/PostgreSQL)
   System (kern, auth)    Rules (xử lý)         Log Server (Elasticsearch, Kafka)
                          Output (đích đến)     Remote Log Server (UDP/TCP)
```

---

## 3. Mức độ Severity

| Mức | Tên       | Ý nghĩa |
|-----|-----------|---------|
| 0   | emerg     | Hệ thống không sử dụng được |
| 1   | alert     | Cần hành động ngay lập tức |
| 2   | crit      | Lỗi nghiêm trọng |
| 3   | err       | Lỗi thông thường |
| 4   | warning   | Cảnh báo |
| 5   | notice    | Thông báo bình thường |
| 6   | info      | Thông tin |
| 7   | debug     | Gỡ lỗi |

## 4. Facility phổ biến

| Facility       | Ý nghĩa |
|----------------|---------|
| auth / authpriv| Xác thực, đăng nhập |
| cron           | Tiến trình cron |
| daemon         | Dịch vụ chạy nền |
| kern           | Kernel |
| mail           | Mail server |
| local0–local7  | Tùy chỉnh người dùng |

**Công thức:** `PRI = Facility × 8 + Severity`

---

## 5. Thực hành với RSYSLOG

### 5.1. Cài đặt và kiểm tra
```bash
# Ubuntu / Debian
sudo apt update && sudo apt install rsyslog -y

# CentOS / RHEL
sudo yum install rsyslog -y

# Kiểm tra & bật dịch vụ
sudo systemctl status rsyslog
sudo systemctl enable rsyslog
```
- File cấu hình chính: `/etc/rsyslog.conf`
- File log mặc định: `/var/log/syslog` (Debian/Ubuntu), `/var/log/messages` (RHEL/CentOS)

### 5.2. Cấu hình cơ bản

**Ghi log theo facility & severity:**
```conf
# /etc/rsyslog.d/50-default.conf
auth,authpriv.*     /var/log/auth.log
cron.*              /var/log/cron.log
kern.warning        /var/log/kern_warn.log
```

**Gửi log đến máy chủ từ xa:**
```conf
# UDP
*.*  @192.168.1.10:514

# TCP (an toàn hơn)
*.*  @@192.168.1.10:514
```
> Lưu ý mở firewall cho port 514 (UDP/TCP) ở client.

### 5.3. Cấu hình server nhận log

```conf
# /etc/rsyslog.d/10-remote.conf
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Template lưu log theo từng host
template(name="RemoteHost" type="string"
         string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")
*.* ?RemoteHost
& stop
```
Khởi động lại dịch vụ:
```bash
sudo systemctl restart rsyslog
```

### 5.4. Kiểm tra

**Trên client (gửi log test):**
```bash
logger -p local0.info "Test log từ client (hostname)"
```

**Trên server (xem log đã nhận):**
```bash
ls /var/log/remote/
tail -f /var/log/remote/<hostname>/<program>.log
```

Ví dụ kết quả:
```
2025-05-20T10:30:15 client01 Test log từ client01
2025-05-20T10:34:20 client02 Test log từ client02
```

---

## 6. Một số lệnh hữu ích

| Lệnh | Công dụng |
|------|-----------|
| `logger "message"`             | Ghi log thủ công |
| `rsyslogd -N1`                 | Kiểm tra cú pháp file cấu hình |
| `systemctl restart rsyslog`    | Khởi động lại dịch vụ |
| `tail -f /var/log/syslog`      | Xem log realtime |
| `journalctl -u rsyslog`        | Xem log của dịch vụ rsyslog |

---

## 7. Tóm tắt

- **Syslog**: chuẩn ghi & truyền log.
- **Rsyslog**: daemon mạnh mẽ, linh hoạt, hỗ trợ gửi log tập trung qua mạng.
- Kết hợp **PRI = Facility × 8 + Severity** để phân loại log.
- Triển khai mô hình **client → server** giúp tập trung, giám sát và phân tích log hiệu quả.

> **TIP:** Trong môi trường thực tế, nên dùng **TCP/TLS (port 6514)** thay vì UDP để đảm bảo log không bị mất và được mã hóa khi truyền qua mạng.
