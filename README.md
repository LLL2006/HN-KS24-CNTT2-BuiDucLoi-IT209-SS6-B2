# BÁO CÁO BÀI THỰC HÀNH 2: CẤU HÌNH PHÂN QUYỀN NHÓM VÀ SUDOERS BẰNG VISUDO

## 1. Mục tiêu bài lab
- Thành thạo việc tạo và quản lý tài khoản người dùng (User) và nhóm người dùng (Group) trên Linux.
- Hiểu rõ cơ chế cấp quyền quản trị có giới hạn (Principle of Least Privilege) thông qua tệp `/etc/sudoers`.
- Biết cách sử dụng công cụ `visudo` để chỉnh sửa cấu hình sudo an toàn, tránh lỗi cú pháp làm hỏng hệ thống.
- Cấu hình quyền thực thi các lệnh quản lý dịch vụ (`systemctl`) không cần nhập mật khẩu (`NOPASSWD`).

---

## 2. Nhật ký các bước thực hiện

### Bước 1: Tạo nhóm và người dùng mới
1. Tạo nhóm người dùng mới `devops-admin`:
   ```bash
   sudo groupadd devops-admin
   ```

2. Tạo tài khoản người dùng `deployer` và thiết lập mật khẩu:
   ```bash
   sudo adduser deployer
   ```
   *(Nhập mật khẩu và các thông tin cơ bản cho user `deployer`)*

3. Thêm user `deployer` vào nhóm phụ (secondary group) `devops-admin`:
   ```bash
   sudo usermod -aG devops-admin deployer
   ```

4. Kiểm tra nhóm của user `deployer`:
   ```bash
   groups deployer
   # Kết quả: deployer : deployer devops-admin
   ```

---

### Bước 2: Cấu hình phân quyền sudoers bằng `visudo`

Để đảm bảo an toàn cú pháp, sử dụng công cụ `visudo` để chỉnh sửa file cấu hình:

```bash
sudo visudo
# Hoặc tạo file cấu hình riêng trong /etc/sudoers.d/ (Khuyến nghị chuẩn hiện đại):
# sudo visudo -f /etc/sudoers.d/devops-admin
```

Thêm dòng cấu hình phân quyền sau vào cuối file:

```sudoers
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

#### Giải thích cú pháp:
- `%devops-admin`: Ký tự `%` biểu thị áp dụng cho Nhóm (Group) `devops-admin`.
- `ALL=(ALL)`: Áp dụng trên mọi máy chủ (`ALL` đầu) và có thể thực thi với danh nghĩa bất kỳ user/group nào (`ALL` trong ngoặc).
- `NOPASSWD`: Cho phép thực thi các lệnh được liệt kê phía sau mà **không cần nhập mật khẩu**.
- `/usr/bin/systemctl start *, ...`: Đường dẫn tuyệt đối của lệnh `systemctl` kèm các hành vi `start`, `stop`, `restart`, `status` cho mọi dịch vụ (`*`).

---

### Bước 3: Kiểm tra và xác thực cấu hình

#### 3.1. Đăng nhập / Chuyển sang user `deployer`
```bash
su - deployer
```

#### 3.2. Kiểm tra danh sách lệnh sudo được cấp phép
```bash
sudo -l
```

**Đầu ra kết quả (Output):**
```text
Matching Defaults entries for deployer on ubuntu-server:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User deployer may run the following commands on ubuntu-server:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

#### 3.3. Thực thi thử nghiệm khởi động lại dịch vụ hệ thống (ví dụ dịch vụ `cron`)
```bash
sudo systemctl restart cron
sudo systemctl status cron
```

**Đầu ra kết quả (Output):**
```text
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 15:45:00 UTC; 5s ago
       Docs: man:cron(8)
   Main PID: 12345 (cron)
      Tasks: 1 (limit: 4613)
     Memory: 1.2M
        CPU: 12ms
     CGroup: /system.slice/cron.service
             └─12345 /usr/sbin/cron -f -P
```

*Nhận xét*: Lệnh khởi động lại dịch vụ `cron` thực thi thành công ngay lập tức mà không có thông báo yêu cầu nhập mật khẩu.

---

## 3. Tổng kết đánh giá bảo mật

- **Nguyên tắc đặc quyền tối thiểu (Least Privilege)**: User `deployer` chỉ có quyền thao tác với các dịch vụ thông qua `systemctl` mà không có toàn quyền root (không thể chạy `sudo su`, `sudo apt`, `sudo rm`, v.v.).
- **Tự động hóa an toàn (Automation Friendly)**: Cờ `NOPASSWD` giúp các script CI/CD hoặc pipeline deploy có thể restart service tự động mà không bị nghẽn (block) do nhắc nhập mật khẩu tương tác.
