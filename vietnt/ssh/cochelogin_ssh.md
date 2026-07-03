# Cơ chế SSH login vào Linux — Các file được thực thi

## Luồng tổng quan

```
sshd  →  PAM stack  →  SSH hooks  →  Shell startup files
```

### Diagram: Toàn bộ luồng SSH login

```mermaid
flowchart TD
    A[Client: ssh alice@server] -->|TCP port 22| B[sshd daemon lắng nghe]
    B --> C[/etc/ssh/sshd_config<br/>Đọc cấu hình/]
    C --> D{Có Banner?}
    D -->|Có| E[/etc/issue.net<br/>Hiện banner cảnh báo/]
    D -->|Không| F
    E --> F[PAM: /etc/pam.d/sshd]

    F --> G[auth<br/>pam_unix.so + pam_env.so]
    G --> G1[(/etc/passwd<br/>/etc/shadow)]
    G --> G2[(/etc/environment)]
    G --> H{Auth OK?}
    H -->|Fail| X[Đóng kết nối]
    H -->|OK| I[account<br/>pam_nologin.so]

    I --> I1[(/etc/nologin?)]
    I --> J{Account OK?}
    J -->|Fail| X
    J -->|OK| K[session<br/>pam_limits + pam_lastlog + pam_motd]

    K --> K1[(/etc/security/limits.conf)]
    K --> K2[(/var/log/lastlog)]
    K --> K3[(/etc/motd<br/>/etc/update-motd.d/*)]

    K --> L{Có ~/.ssh/rc?}
    L -->|Có| L1[Chạy ~/.ssh/rc]
    L -->|Không| L2[Chạy /etc/ssh/sshrc<br/>nếu có]
    L1 --> M
    L2 --> M

    M[Fork login shell: -bash]
    M --> N[/etc/profile]
    N --> N1[/etc/profile.d/*.sh]
    N1 --> O[~/.bash_profile<br/>hoặc ~/.bash_login<br/>hoặc ~/.profile]
    O --> P[~/.bashrc<br/>nếu được source]
    P --> Q([Prompt sẵn sàng<br/>alice@server:~$])

    Q -.logout.-> R[~/.bash_logout]
    R --> S[/etc/bash.bash_logout]
    S --> T[PAM session close]
    T --> U[sshd đóng kết nối]

    style A fill:#e1f5ff
    style Q fill:#c8e6c9
    style X fill:#ffcdd2
    style F fill:#fff9c4
    style M fill:#f8bbd0
```

### Diagram: PAM stack chi tiết

```mermaid
flowchart LR
    SSHD[sshd] --> PAM[/etc/pam.d/sshd]

    PAM --> AUTH[auth group]
    PAM --> ACCT[account group]
    PAM --> SESS[session group]
    PAM --> PWD[password group]

    AUTH --> A1[pam_unix.so<br/>check password]
    AUTH --> A2[pam_env.so<br/>load env vars]
    AUTH --> A3[pam_google_authenticator<br/>2FA optional]

    ACCT --> B1[pam_nologin.so<br/>check /etc/nologin]
    ACCT --> B2[pam_access.so<br/>check IP/time rules]

    SESS --> C1[pam_limits.so<br/>apply ulimit]
    SESS --> C2[pam_lastlog.so<br/>Last login banner]
    SESS --> C3[pam_motd.so<br/>show MOTD]

    PWD --> D1[pam_pwquality.so<br/>khi đổi password]

    style PAM fill:#fff9c4
    style AUTH fill:#ffe0b2
    style ACCT fill:#e1bee7
    style SESS fill:#c5e1a5
    style PWD fill:#b3e5fc
```

### Diagram: Interactive vs Non-interactive

```mermaid
flowchart TD
    START[ssh command] --> CHECK{Có command<br/>đi kèm?}

    CHECK -->|Không:<br/>ssh user@host| INT[Interactive Login Shell]
    CHECK -->|Có:<br/>ssh user@host 'ls'| NON[Non-interactive Shell]

    INT --> I1[/etc/profile]
    I1 --> I2[/etc/profile.d/*.sh]
    I2 --> I3[~/.bash_profile]
    I3 --> I4[~/.bashrc<br/>nếu được source]
    I4 --> I5([Prompt hiện])

    NON --> N1[Chỉ đọc ~/.bashrc]
    N1 --> N2{Đầu file có<br/>PS1 check?}
    N2 -->|Có| N3[Thoát sớm<br/>không load config]
    N2 -->|Không| N4[Load hết .bashrc]
    N3 --> N5[Chạy command]
    N4 --> N5
    N5 --> N6([Thoát])

    style INT fill:#c8e6c9
    style NON fill:#ffe0b2
    style I5 fill:#81c784
    style N6 fill:#ffb74d
```

---

## 1. SSHD + PAM (trước khi shell chạy)

| File | Vai trò |
|------|--------|
| `/etc/ssh/sshd_config` | Cấu hình sshd |
| `/etc/pam.d/sshd` | PAM stack cho SSH |
| `/etc/pam.d/common-*` (Debian) hoặc `/etc/pam.d/system-auth` (RHEL) | Module PAM dùng chung |
| `/etc/environment` | `pam_env.so` đọc → set biến môi trường hệ thống |
| `/etc/security/limits.conf` | `pam_limits.so` áp ulimit |
| `/etc/nologin` | Nếu tồn tại → chặn login (trừ root) |
| `/etc/issue.net` | Banner trước khi auth |
| `/etc/motd`, `/etc/update-motd.d/*` | Banner sau khi login (pam_motd.so) |
| `/var/log/auth.log` (Debian) / `/var/log/secure` (RHEL) | Log sshd + PAM |

---

## 2. SSH hooks (giữa PAM và shell)

| File | Vai trò |
|------|--------|
| `/etc/ssh/sshrc` | Script system-wide, chạy trước shell |
| `~/.ssh/rc` | Script per-user, chạy trước shell (nếu có, `sshrc` bị bỏ qua) |
| `~/.ssh/authorized_keys` (option `command=`) | Ép chạy command thay vì shell |

---

## 3. Shell startup — Bash (mặc định)

**Interactive login shell (SSH bình thường):**

```
1. /etc/profile
2. /etc/profile.d/*.sh          (được /etc/profile source)
3. Một trong: ~/.bash_profile → ~/.bash_login → ~/.profile
4. ~/.bashrc                     (nếu ~/.bash_profile source nó)
5. Logout: ~/.bash_logout
```

**Non-interactive (chạy lệnh: `ssh user@host command`):**

- Chỉ đọc `~/.bashrc`
- **KHÔNG** chạy `/etc/profile`, `~/.bash_profile`

---

## 4. Shell startup — Zsh

```
/etc/zshenv   → ~/.zshenv
/etc/zprofile → ~/.zprofile
/etc/zshrc    → ~/.zshrc
/etc/zlogin   → ~/.zlogin
Logout:  ~/.zlogout → /etc/zlogout
```

---

## 5. Thứ tự đầy đủ cho SSH interactive login (bash)

```
1. sshd accept connection
2. /etc/pam.d/sshd → PAM stack
   ├─ auth:    pam_unix, pam_env       (/etc/environment)
   ├─ account: pam_nologin             (/etc/nologin)
   ├─ session: pam_limits              (/etc/security/limits.conf)
   │           pam_lastlog, pam_motd   (/etc/motd)
3. /etc/ssh/sshrc  hoặc  ~/.ssh/rc
4. Login shell start
5. /etc/profile → /etc/profile.d/*.sh
6. ~/.bash_profile → ~/.bashrc
7. Prompt ready
```

---

## 6. Điều gì xảy ra khi bạn gõ `ssh user@server`?

Hãy tưởng tượng bạn vừa gõ `ssh alice@server.com` và nhấn Enter. Đây là câu chuyện đằng sau:

### Bước 1 — Bắt tay kết nối
Máy client của bạn gửi yêu cầu tới cổng **22** của server. Trên server, có một tiến trình tên là **`sshd`** (SSH Daemon) đang chạy nền và lắng nghe cổng này 24/7. Nó nhận cuộc gọi, mở file **`/etc/ssh/sshd_config`** để biết mình được cấu hình thế nào (có cho phép root login không? Bắt buộc dùng key hay password? Có bật PAM không?...).

### Bước 2 — Hiện banner cảnh báo (nếu có)
Nếu trong `sshd_config` có dòng `Banner /etc/issue.net`, server sẽ đọc file **`/etc/issue.net`** và in ra cho bạn xem **trước khi** bạn nhập gì cả. Thường là cảnh báo pháp lý kiểu "Hệ thống này chỉ dành cho người được ủy quyền...".

### Bước 3 — Xác thực (Authentication)
Đây là lúc server hỏi "bạn là ai?". `sshd` giao việc này cho **PAM** thông qua file **`/etc/pam.d/sshd`**. File này giống như một checklist — nó bảo PAM chạy lần lượt các module để kiểm tra:

- **`pam_unix.so`** đọc `/etc/passwd` (danh sách user) và `/etc/shadow` (password đã hash) để kiểm tra password bạn nhập. Nếu bạn dùng SSH key thì `sshd` tự xử lý bằng cách đối chiếu với **`~/.ssh/authorized_keys`** trong home directory của user đích.
- **`pam_env.so`** đọc **`/etc/environment`** và nạp các biến môi trường hệ thống (như `LANG`, `PATH` cơ bản).

### Bước 4 — Kiểm tra tài khoản (Account)
Xác thực đúng chưa đủ. PAM tiếp tục hỏi "tài khoản này có được phép login lúc này không?":

- **`pam_nologin.so`** kiểm tra xem có file **`/etc/nologin`** tồn tại không. Nếu có (thường admin tạo lúc bảo trì), mọi user thường bị chặn, chỉ root vào được.
- Các module khác có thể kiểm tra: account hết hạn chưa? Đang giờ được phép login không? IP có bị block không?

### Bước 5 — Chuẩn bị session
Xong phần kiểm tra, giờ chuẩn bị "không gian làm việc" cho bạn:

- **`pam_limits.so`** đọc **`/etc/security/limits.conf`** và áp giới hạn tài nguyên: bạn được mở tối đa bao nhiêu file, chạy bao nhiêu process, dùng bao nhiêu RAM...
- **`pam_lastlog.so`** đọc **`/var/log/lastlog`** và in ra dòng "Last login: ..." quen thuộc, đồng thời ghi lại lần login này.
- **`pam_motd.so`** in **MOTD** — banner chào mừng lấy từ **`/etc/motd`** (tĩnh) cộng với output các script trong **`/etc/update-motd.d/`** (động, ví dụ hiện số update chờ cài, tải hệ thống...).

### Bước 6 — SSH hook (tùy chọn)
Ngay trước khi shell khởi động, `sshd` kiểm tra:
- **`~/.ssh/rc`** — nếu user có file này, nó sẽ được chạy (script riêng cho user).
- **`/etc/ssh/sshrc`** — nếu user không có `~/.ssh/rc` thì file system-wide này chạy thay.

Hai file này ít dùng nhưng hữu ích để log, thông báo Slack, hoặc setup gì đó đặc biệt trước khi user vào shell.

### Bước 7 — Khởi động Shell (login shell)
Giờ tới phần "giao quyền điều khiển" cho user. `sshd` fork một tiến trình shell (thường là `bash`) làm **login shell** — báo hiệu bằng dấu `-` ở đầu tên: `-bash`. Shell này lần lượt đọc:

1. **`/etc/profile`** — script bash system-wide, chạy cho **mọi user** khi login. Thường set `PATH`, `PS1` mặc định, umask...
2. **`/etc/profile.d/*.sh`** — được `/etc/profile` tự động source. Đây là chỗ các package cài thêm bỏ script riêng vào (ví dụ `bash_completion.sh`, `nvm.sh`). Cách này gọn hơn là sửa trực tiếp `/etc/profile`.
3. **`~/.bash_profile`** (hoặc `~/.bash_login`, hoặc `~/.profile` — bash tìm cái nào có trước) — file **cá nhân** của user, muốn set biến riêng, alias riêng thì để ở đây.
4. **`~/.bashrc`** — **KHÔNG** tự động chạy cho login shell! Nhưng theo thông lệ, `~/.bash_profile` sẽ có dòng `[[ -f ~/.bashrc ]] && . ~/.bashrc` để source nó. `.bashrc` là chỗ đặt alias, function, prompt tùy chỉnh.

### Bước 8 — Sẵn sàng
Sau tất cả, bạn thấy prompt `alice@server:~$` — hệ thống đã sẵn sàng nhận lệnh. Cả quá trình trên diễn ra trong vài trăm milliseconds.

### Khi bạn logout
Gõ `exit` hoặc Ctrl+D → shell chạy **`~/.bash_logout`** (dọn dẹp cá nhân) → **`/etc/bash.bash_logout`** (dọn dẹp system-wide) → PAM `session close` (ghi log, giải phóng resource) → `sshd` đóng kết nối.

### Trường hợp đặc biệt: `ssh user@server "ls"`
Khi bạn chạy lệnh trực tiếp không mở shell tương tác, luồng ngắn hơn nhiều:
- Vẫn qua PAM đầy đủ (auth + account + session).
- **KHÔNG** chạy `/etc/profile`, `~/.bash_profile`, `~/.bashrc` chuẩn — chỉ đọc `~/.bashrc` (và thường phần đầu `.bashrc` có dòng `[ -z "$PS1" ] && return` để thoát sớm nếu không interactive).
- Chạy lệnh → in output → thoát ngay.

Đây là lý do nhiều người bị lỗi "biến môi trường không có" khi chạy `ssh host command` — vì `/etc/profile` không được đọc.

---

## 7. Giải thích các thuật ngữ

### PAM (Pluggable Authentication Modules)
Hệ thống xác thực **module hóa** của Linux. Thay vì mỗi ứng dụng (sshd, login, sudo, su...) tự viết code xác thực, chúng gọi PAM. PAM đọc file cấu hình tương ứng (vd `/etc/pam.d/sshd`) rồi lần lượt gọi các module `.so` để làm việc.

**4 nhóm chức năng (management groups):**
| Nhóm | Nhiệm vụ |
|------|---------|
| `auth` | Xác thực user (kiểm tra password, key, 2FA...) |
| `account` | Kiểm tra tài khoản còn hợp lệ không (hết hạn? bị khóa? giờ được login?) |
| `session` | Chuẩn bị/dọn dẹp session (mount home, set limits, ghi log, hiện MOTD) |
| `password` | Đổi mật khẩu (độ mạnh, lịch sử) |

**Các module .so phổ biến:**
- `pam_unix.so` — xác thực bằng `/etc/passwd` + `/etc/shadow`
- `pam_env.so` — nạp biến môi trường từ `/etc/environment`
- `pam_limits.so` — áp giới hạn từ `/etc/security/limits.conf` (số process, RAM, file mở...)
- `pam_nologin.so` — chặn login thường nếu có `/etc/nologin`
- `pam_lastlog.so` — hiện "Last login: ..." và ghi vào `/var/log/lastlog`
- `pam_motd.so` — in Message Of The Day
- `pam_google_authenticator.so` — 2FA TOTP (nếu cài thêm)

### SSHD (SSH Daemon)
Tiến trình chạy nền lắng nghe cổng 22 (mặc định), nhận kết nối SSH. Nó **không tự xác thực** — nó ủy quyền cho PAM (nếu `UsePAM yes`) hoặc dùng cơ chế nội tại (public key).

### Login shell vs Non-login shell
- **Login shell**: shell đầu tiên khi user "đăng nhập" (SSH, tty console). Đọc `/etc/profile` và `~/.bash_profile`. Nhận diện bằng dấu `-` đầu argv[0] (vd `-bash`).
- **Non-login shell**: mở terminal mới sau khi đã login (vd `bash` gõ trong terminal). Chỉ đọc `~/.bashrc`.

### Interactive vs Non-interactive shell
- **Interactive**: có prompt, chờ user gõ. (SSH bình thường)
- **Non-interactive**: chạy script/lệnh rồi thoát. (Vd `ssh user@host "ls"`)

→ Kết hợp lại có 4 loại, mỗi loại đọc bộ file khác nhau. SSH login = **interactive login shell**.

### MOTD (Message Of The Day)
Banner in ra khi login thành công. Nội dung lấy từ `/etc/motd` (tĩnh) và `/run/motd.dynamic` (động, sinh từ script trong `/etc/update-motd.d/`).

### Banner (issue.net)
Chuỗi hiện ra **trước khi** nhập password — thường dùng để cảnh báo pháp lý ("Unauthorized access prohibited..."). Cấu hình bằng `Banner /etc/issue.net` trong `sshd_config`.

### ForceCommand
Option trong `sshd_config` hoặc `authorized_keys`. Ép user chạy 1 command duy nhất thay vì shell tự do — dùng cho tài khoản backup, git-only, rsync-only.

### ulimit / limits.conf
Giới hạn tài nguyên per-process: số file mở tối đa, số process, RAM, kích thước file... `pam_limits.so` đọc `/etc/security/limits.conf` và áp dụng lúc login.

### /etc/environment vs /etc/profile
- `/etc/environment` — file **KEY=VALUE** đơn giản, đọc bởi PAM. Không phải script, không có logic.
- `/etc/profile` — **script bash**, chạy lệnh, có if/for, source file khác. Chỉ chạy cho login shell.

### `.so` file
Shared Object — thư viện chia sẻ trên Linux (tương đương `.dll` trên Windows). PAM module là các file `.so` trong `/lib/x86_64-linux-gnu/security/` (Debian) hoặc `/usr/lib64/security/` (RHEL).

---

## 8. Debug

```bash
ssh -v user@host              # verbose client
bash -x -l                    # trace login shell startup
journalctl -u ssh             # log sshd
tail -f /var/log/auth.log     # log auth (Debian/Ubuntu)
```
