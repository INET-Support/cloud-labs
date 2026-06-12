# 1. Cài đặt & Kiểm tra rsyslog
![alt text](caidat_rsyslog-1.png)

# 2. Cấu hình rsyslog conf
![alt text](cauhinh_rsyslogconf-1.png)

# 2. Gửi log test từ client
## Client
vietnt@client-01:~$ logger "Hello from client"
vietnt@client-01:~$ logger -t nginx "Access test"

## Server
vietnt@server:~$ grep -R "Hello from client" /var/log/remote
/var/log/remote/client-01/vietnt.log:2026-06-12T05:03:19+00:00 client-01 vietnt: Hello from client
vietnt@server:~$ cat /var/log/remote/client-01/nginx.log
2026-06-12T06:05:09+00:00 client-01 nginx: Access test

# 3. Ví dụ cấu hình một số loại log
# ==========================
# SECURITY LOGS
# ==========================
# SSH, sudo, su, PAM
auth,authpriv.* @@192.168.122.50:514

# ==========================
# SYSTEM ERRORS
# ==========================
# Kernel warning/error
kern.warning @@192.168.122.50:514

# Tất cả lỗi từ mọi facility
*.err @@192.168.122.50:514

# ==========================
# IMPORTANT EVENTS
# ==========================
# Dịch vụ hệ thống
daemon.warning @@192.168.122.50:514

# Mail server (nếu có)
mail.warning @@192.168.122.50:514

# Cron lỗi/cảnh báo
cron.warning @@192.168.122.50:514

# ==========================
# CUSTOM APPLICATIONS
# ==========================
# local0-local7 dành cho ứng dụng tự phát triển
local0.* @@192.168.122.50:514
local1.* @@192.168.122.50:514
local2.* @@192.168.122.50:514
local3.* @@192.168.122.50:514

# ==========================
# PREVENT DUPLICATE PROCESSING
# ==========================
& stop

# 4. Thử setup parse log json

> **Topology:**
> - Server log (nhận log): `192.168.122.50`
> - Client (gửi log): `192.168.122.52` — hostname `client-01`

## 4.1. Cấu hình SERVER (192.168.122.50)
### Tạo config nhận + parse JSON
File `/etc/rsyslog.d/30-server-receive-json.conf`:
![alt text](image-2.png)

### Mở firewall + restart
```bash
sudo ufw allow 514/tcp
sudo rsyslogd -N1
sudo systemctl restart rsyslog
sudo ss -ltnp | grep 514     # LISTEN 0.0.0.0:514
```

---

## 4.2. Cấu hình CLIENT (192.168.122.52)
### Tạo config forward JSON
File `/etc/rsyslog.d/90-client-forward-json.conf`:
![alt text](image-1.png)

### Validate + restart
```bash
sudo rsyslogd -N1
sudo systemctl restart rsyslog
```

---

## 4.3. Test gửi log

### Trên CLIENT
vietnt@client-01:~$ logger -t nginx "GET /api/users 200 OK"

### Trên SERVER — verify
vietnt@server:~$ sudo cat /var/log/remote/client-01/nginx.json | jq .
{
  "@timestamp": "2026-06-12T07:11:49.289550+00:00",
  "host": "client-01",
  "severity": "notice",
  "facility": "user",
  "app": "nginx",
  "pid": "-",
  "msg": " GET /api/users 200 OK"
}

![alt text](image.png)