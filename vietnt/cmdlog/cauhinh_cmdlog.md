# Transcript cấu hình CMD Log — Ubuntu 24.04

**Server:** `vietnt@server`
**Ngày thực hiện:** 2026-07-03
**Kết quả:** Thành công — log mọi lệnh bash vào `/var/log/cmdlog/cmd.log`

Đây là ghi chép nguyên văn các lệnh đã chạy theo thứ tự đúng (đã loại các bước sai / debug). Copy-paste theo thứ tự này trên server mới sẽ chạy được ngay.

---

## Bước 1 — Đảm bảo rsyslog chạy

```
vietnt@server:~$ sudo systemctl enable --now rsyslog
```

---

## Bước 2 — Tạo thư mục log (owner phải là syslog)

```
vietnt@server:~$ sudo mkdir -p /var/log/cmdlog
vietnt@server:~$ sudo chown syslog:adm /var/log/cmdlog
vietnt@server:~$ sudo chmod 0755 /var/log/cmdlog
```

Kiểm tra:

```
vietnt@server:~$ ls -la /var/log/cmdlog/
total 8
drwxr-xr-x  2 syslog adm    4096 Jul  3 20:40 .
drwxr-xr-x 13 root   syslog 4096 Jul  3 20:32 ..
```

---

## Bước 3 — Tạo hook PROMPT_COMMAND

```
vietnt@server:~$ sudo tee /etc/profile.d/cmdlog.sh > /dev/null <<'EOF'
[ -z "$BASH_VERSION" ] && return
[[ $- != *i* ]] && return
export HISTTIMEFORMAT="%s "
__cmd_audit() {
    local last cmd src
    last=$(history 1)
    cmd=$(echo "$last" | sed -E 's/^\s*[0-9]+\s+[0-9]+\s+//')
    [ "$cmd" = "$__LAST_CMD" ] && return
    __LAST_CMD="$cmd"
    src="${SSH_CLIENT%% *}"
    src="${src:-local}"
    logger -p local6.info -t bash_cmd "user=$USER tty=$(tty 2>/dev/null) pid=$$ src=$src pwd=$PWD cmd=\"$cmd\""
}
PROMPT_COMMAND='__cmd_audit; history -a'
EOF
vietnt@server:~$ sudo chmod 0644 /etc/profile.d/cmdlog.sh
```

Verify syntax:

```
vietnt@server:~$ bash -n /etc/profile.d/cmdlog.sh && echo "SYNTAX OK"
SYNTAX OK
```

---

## Bước 4 — Cấu hình rsyslog tách log

```
vietnt@server:~$ sudo tee /etc/rsyslog.d/50-cmdlog.conf > /dev/null <<'EOF'
if $programname == 'bash_cmd' then {
    action(type="omfile"
           file="/var/log/cmdlog/cmd.log"
           fileCreateMode="0640"
           fileOwner="root"
           fileGroup="adm")
    stop
}
EOF
```

---

## Bước 5 — Bật journald forward sang rsyslog (Ubuntu 24.04 bắt buộc)

```
vietnt@server:~$ sudo sed -i 's/^#\?ForwardToSyslog=.*/ForwardToSyslog=yes/' /etc/systemd/journald.conf
vietnt@server:~$ grep ForwardToSyslog /etc/systemd/journald.conf
ForwardToSyslog=yes
```

---

## Bước 6 — Restart journald + rsyslog

```
vietnt@server:~$ sudo systemctl restart systemd-journald
vietnt@server:~$ sudo systemctl restart rsyslog
vietnt@server:~$ systemctl is-active rsyslog
active
```

---

## Bước 7 — Nạp hook cho session hiện tại (hoặc logout/SSH lại)

```
vietnt@server:~$ source /etc/profile.d/cmdlog.sh
vietnt@server:~$ echo "PROMPT_COMMAND=[$PROMPT_COMMAND]"
PROMPT_COMMAND=[__cmd_audit; history -a]
```

---

## Bước 8 — Test kiểm chứng

```
vietnt@server:~$ whoami
vietnt
vietnt@server:~$ pwd
/home/vietnt
vietnt@server:~$ echo "hello cmdlog"
hello cmdlog
```

Xem log:

```
vietnt@server:~$ cd /var/log/cmdlog
vietnt@server:/var/log/cmdlog$ sudo cat cmd.log
2026-07-03T20:41:00.978220+07:00 server bash_cmd: user=vietnt tty=/dev/pts/0 pid=183490 src=10.0.14.3 pwd=/home/vietnt cmd="whoami"
2026-07-03T20:41:14.823551+07:00 server bash_cmd: user=vietnt tty=/dev/pts/0 pid=183490 src=10.0.14.3 pwd=/home/vietnt cmd="pwd"
2026-07-03T20:43:21.952480+07:00 server bash_cmd: user=vietnt tty=/dev/pts/0 pid=183490 src=10.0.14.3 pwd=/home/vietnt cmd="echo \"hello cmdlog\""
```

---

## Bước 9 — Verify sau khi SSH lại (kiểm tra persistent)

```
vietnt@server:~$ exit
```

Sau khi SSH lại:

```
vietnt@server:~$ echo "PROMPT_COMMAND=[$PROMPT_COMMAND]"
PROMPT_COMMAND=[__cmd_audit; history -a]
vietnt@server:~$ whoami
vietnt
vietnt@server:~$ sudo tail -n 3 /var/log/cmdlog/cmd.log
2026-07-03T20:43:25.437601+07:00 server bash_cmd: user=vietnt tty=/dev/pts/0 pid=183490 src=10.0.14.3 pwd=/home/vietnt cmd="whoami"
```

Hook vẫn hoạt động sau logout → **cấu hình persistent**.

---

## Tổng kết files đã tạo/sửa

| File | Vai trò |
|---|---|
| `/etc/profile.d/cmdlog.sh` | Hook PROMPT_COMMAND, chạy `logger` mỗi lệnh |
| `/etc/rsyslog.d/50-cmdlog.conf` | Rule rsyslog: tag `bash_cmd` → file riêng |
| `/etc/systemd/journald.conf` | Đổi `ForwardToSyslog=no` → `yes` |
| `/var/log/cmdlog/` | Thư mục chứa log, owner `syslog:adm` |
| `/var/log/cmdlog/cmd.log` | File log CMD |

## Rollback (nếu cần gỡ)

```
vietnt@server:~$ sudo rm /etc/profile.d/cmdlog.sh
vietnt@server:~$ sudo rm /etc/rsyslog.d/50-cmdlog.conf
vietnt@server:~$ sudo sed -i 's/^ForwardToSyslog=yes/#ForwardToSyslog=no/' /etc/systemd/journald.conf
vietnt@server:~$ sudo systemctl restart systemd-journald rsyslog
vietnt@server:~$ sudo rm -rf /var/log/cmdlog
```

---

## Lưu ý quan trọng (đã bị dính trong quá trình setup)

1. **Owner thư mục phải là `syslog`**, không phải `root`. Nếu để `root:adm` chmod 0750 → rsyslog báo `omfile suspended` không ghi file được.
2. **Ubuntu 24.04 mặc định journald không forward** sang rsyslog. Phải bật `ForwardToSyslog=yes` mới có log.
3. **Session hiện tại** cần `source` hoặc SSH lại thì hook mới nạp. Terminal client (như Termius) có thể override `PROMPT_COMMAND` — kiểm tra bằng `echo $PROMPT_COMMAND`.
4. Config VictoriaLogs (`95-victorialogs.conf`) báo lỗi thiếu module `omhttp` là **vấn đề riêng biệt**, không ảnh hưởng CMD log.
