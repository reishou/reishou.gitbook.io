---
icon: lock-keyhole
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/getting-started/essential-initial-security-hardening
---

# Bảo Mật Cơ Bản

Bài viết này hướng dẫn các bước bảo mật cơ bản nhưng cực kỳ hiệu quả cho một VPS mới, giúp giảm đáng kể nguy cơ bị tấn công tự động (brute-force, scanning...).

**Điều kiện tiên quyết (bạn cần hoàn thành trước khi làm theo bài này):**

* Đã tạo user không phải root có quyền sudo (ví dụ: `vps-user`).
* Đã thiết lập SSH key cho user `vps-user` và **có thể đăng nhập SSH bình thường** bằng key (không dùng mật khẩu). Nếu chưa làm được 2 điều trên → hãy dừng lại và xử lý trước, vì các bước sau sẽ khóa root và cấm đăng nhập bằng mật khẩu.

## Cập nhật hệ thống & cài gói cơ bản

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget vim ufw fail2ban apt-listchanges
```

## Vô hiệu hóa đăng nhập root & chỉ cho phép SSH key

Sao lưu trước khi sửa:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%F)
sudo nano /etc/ssh/sshd_config
```

Sửa/chỉnh các dòng sau (comment hoặc thay đổi giá trị):

```ini
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# Giới hạn số lần thử đăng nhập (tùy chọn nhưng rất nên có)
MaxAuthTries 4

# Chỉ cho phép user cụ thể (rất khuyến khích)
AllowUsers vps-user
```

Kiểm tra và áp dụng:

```bash
sudo sshd -t          # Kiểm tra cú pháp, cực kỳ quan trọng!
sudo systemctl restart sshd
```

**Mẹo an toàn**: Mở thêm 1 terminal SSH bằng `vps-user` song song trước khi restart để đề phòng lỗi.

## (Khuyến nghị) Đổi port SSH mặc định

Giảm mạnh số lượng bot quét cổng 22.

Trong `/etc/ssh/sshd_config`:

```ini
# Port 22
Port 2204    # chọn cổng > 1024, tránh trùng dịch vụ phổ biến
```

Sau đó:

```bash
sudo sshd -t
sudo systemctl restart sshd
```

Từ lần sau bạn sẽ kết nối bằng lệnh:

```bash
ssh -p 2204 vps-user@123.45.67.89
```

## Thiết lập Firewall với UFW

UFW (Uncomplicated Firewall) là công cụ firewall mặc định và cực kỳ thân thiện trên Ubuntu, giúp bạn dễ dàng quản lý quy tắc iptables mà không cần nhớ syntax phức tạp.

**Nguyên tắc vàng khi dùng UFW trên VPS:**

* Luôn **allow SSH** (hoặc port SSH tùy chỉnh) **trước** khi enable firewall → nếu không bạn sẽ tự khóa chính mình ra ngoài!
* Chính sách mặc định nên là **deny incoming** (chặn hết incoming trừ những gì bạn allow) và **allow outgoing** (cho phép server kết nối ra ngoài bình thường, ví dụ update, DNS, tải package...).

UFW hỗ trợ hai cách chính để mở port: dùng **số port cụ thể** hoặc dùng **tên service/profile**.

**Cách 1 - Dùng port số trực tiếp (cách phổ biến, linh hoạt nhất – đặc biệt khi bạn đã đổi port SSH):**

```bash
# Mở port SSH tùy chỉnh (ví dụ port 2204, chỉ TCP vì SSH dùng TCP)
sudo ufw allow 2204/tcp comment "SSH custom port"

# Hoặc mở cả TCP + UDP (ít dùng cho SSH, nhưng có thể cho dịch vụ khác)
sudo ufw allow 53 comment "DNS - both TCP/UDP"
```

→ Ưu điểm: Rõ ràng, chính xác, đặc biệt khi bạn dùng port không chuẩn (như 2204 thay vì 22). → Nhược điểm: Phải nhớ port + protocol (thường là /tcp), dễ sai nếu port thay đổi.

**Cách 2 - Dùng tên service/profile (rất tiện lợi cho các dịch vụ chuẩn):**

Trước tiên, xem danh sách các profile sẵn có (do package cài đặt tạo):

```bash
sudo ufw app list
# Ví dụ output phổ biến:
# Available applications:
#   Apache
#   Apache Full
#   Apache Secure
#   Nginx Full
#   OpenSSH
#   Postfix
#   ...
```

Sau đó allow bằng tên:

```bash
# Cho SSH (port 22 mặc định)
sudo ufw allow OpenSSH
# hoặc đơn giản hơn (UFW tra cứu /etc/services)
sudo ufw allow ssh

# Cho web server (ví dụ Nginx full: 80 + 443)
sudo ufw allow 'Nginx Full'

# Xem chi tiết một profile (rất hữu ích!)
sudo ufw app info OpenSSH
# Output ví dụ: ports=22/tcp
```

→ Ưu điểm:

* Dễ đọc, dễ quản lý (tên service thay vì số port bí ẩn).
* Tự động xử lý protocol (thường TCP cho SSH, cả TCP/UDP cho HTTP nếu cần).
* Khi xóa rule cũng dễ: `sudo ufw delete allow OpenSSH` (xóa sạch cả IPv4 + IPv6 nếu có).
* Nhiều package (Apache, Nginx, Samba...) tự tạo profile khi cài.

→ Nhược điểm: Chỉ dùng được cho port mặc định của service. Nếu bạn **đổi port SSH sang 2204**, thì `allow ssh` hoặc `allow OpenSSH` sẽ **không mở port 2204** nữa → lúc này bắt buộc phải dùng port số: `allow 2204/tcp`.

**Mẹo thực tế (khuyến nghị cho VPS):**

* Nếu bạn **giữ port SSH mặc định 22** → dùng `sudo ufw allow OpenSSH` hoặc `sudo ufw allow ssh` (rất tiện).
* Nếu bạn **đổi port SSH** (như 2204) → dùng port số: `sudo ufw allow 2204/tcp`.
*   Kết hợp **limit** cho SSH để chống brute-force:<br>

    ```bash
    sudo ufw limit OpenSSH          # Nếu dùng port 22
    # hoặc
    sudo ufw limit 2204/tcp         # Nếu dùng port tùy chỉnh
    ```

Các lệnh cơ bản:

```bash
# Cho phép SSH (dùng port mới nếu đã đổi)
sudo ufw allow 2204/tcp

# Giới hạn rate cho SSH (chống quét nhanh)
sudo ufw limit 2204/tcp comment "SSH rate-limited"

# Chính sách mặc định
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Kích hoạt
sudo ufw enable

# Kiểm tra
sudo ufw status verbose
# Hoặc xem theo số thứ tự (dễ xóa rule sau)
sudo ufw status numbered
```

*   Bật logging nhẹ để theo dõi:<br>

    ```bash
    sudo ufw logging low
    # Xem log: tail -f /var/log/ufw.log
    ```



*   Nếu lỡ cấu hình sai và bị khóa: dùng console của nhà cung cấp VPS để login và reset:<br>

    ```bash
    sudo ufw disable   # hoặc sudo ufw reset (xóa hết rule)
    ```

## Cài đặt & cấu hình Fail2Ban

```bash
sudo systemctl enable fail2ban --now
```

Tạo file cấu hình riêng (không sửa trực tiếp `jail.conf`):

```bash
sudo nano /etc/fail2ban/jail.local
```

Nội dung cơ bản khuyến nghị:

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 4

[sshd]
enabled   = true
port      = 2204           # Đổi nếu bạn đã đổi port SSH
filter    = sshd
logpath   = /var/log/auth.log
maxretry  = 4
bantime   = 86400          # Ban 1 ngày cho SSH
findtime  = 1200
```

* `bantime`: Thời gian bị cấm (giây)
* `findtime`: Khoảng thời gian tính số lần thử sai (giây)
* `maxretry`: Số lần đăng nhập sai tối đa trong `findtime` thì bị ban

Áp dụng:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd   # Kiểm tra jail sshd
```

## Bật tự động cập nhật bảo mật (Unattended Upgrades)

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Chọn Yes khi hỏi về tự động cài bản vá bảo mật
```

Hoặc chỉnh tay (khuyến nghị):

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Bỏ comment (xóa `//`) các dòng sau:

```ini
"${distro_id}:${distro_codename}-security";
"${distro_id}:${distro_codename}-updates";
```

Và đảm bảo file `/etc/apt/apt.conf.d/20auto-upgrades` có nội dung:

Bash

```ini
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

#### Tóm tắt Checklist nhanh (thứ tự ưu tiên)

1. Đảm bảo đăng nhập được bằng `vps-user` + SSH key
2. Disable root + cấm PasswordAuthentication trong `sshd_config`
3. Đổi port SSH (rất khuyến khích)
4. Cấu hình UFW (chỉ mở port cần thiết)
5. Cài & cấu hình Fail2Ban cho SSH
6. Bật unattended-upgrades

Sau khi hoàn thành, VPS của bạn đã ở mức bảo mật cơ bản rất tốt so với trạng thái mặc định từ nhà cung cấp.
