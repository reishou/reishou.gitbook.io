---
icon: rectangle-terminal
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/local-automation/writing-your-first-deployment-script-bash
---

# Thiết lập VPS ban đầu tự động bằng Bash Script

## Yêu cầu trước khi bắt đầu

* Máy local (Mac, Linux, hoặc Windows với WSL/Git Bash) đã cài: git, openssh-client.
* VPS mới, ít nhất 1 CPU – 1-2 GB RAM (tùy stack sau này).
* Thông tin truy cập: IP VPS, root password (hoặc SSH key nếu đã có).
* Hệ điều hành khuyến nghị: Ubuntu 22.04 / 24.04 LTS hoặc Debian 12.

## Chuẩn bị trên VPS – Cài git và clone repository

Đăng nhập vào VPS bằng root từ máy local:

```bash
ssh root@<IP_VPS>
```

Cập nhật hệ thống và cài git nếu chưa có (trên Ubuntu/Debian):

```bash
apt update && apt upgrade -y
apt install git -y
```

Clone repository về VPS (vào thư mục `/root/setup-vps` để dễ quản lý):

```bash
git clone https://github.com/reishou/chomusuke-vps-bash.git /root/setup-vps
cd /root/setup-vps
```

Cấu trúc repo:

* thư mục config/ - chứa các config dành cho các app như laravel, nextjs...
* thư mục `scripts/` – chứa các script `.sh` như `setup-vps.sh`,  `create-non-root-user.sh`, `change-ssh-port.sh`...
* `setup-vps.sh` – script chính chạy toàn bộ quy trình.
* `create-non-root-user.sh` – tạo user non-root.
* `change-ssh-port.sh` – đổi port SSH.
* `setup-local-ssh.sh` – generate và copy SSH key từ local lên VPS.

Cấp quyền chạy scripts:

```bash
chmod +x scripts/*.sh
```

#### Bước 2: Tạo user non-root

Vẫn ở thư mục repo, chạy:

<pre class="language-bash"><code class="lang-bash"><strong>./scripts/create-non-root-user.sh
</strong></code></pre>

Script sẽ:

* Hỏi tên user mới (ví dụ: reishou, dev...).
* Tạo user và đặt password mạnh.
* Thêm user vào group sudo.
* Test quyền sudo.

Sau khi hoàn tất, logout khỏi root:

```bash
exit
```

Đăng nhập lại bằng user mới:

```bash
ssh <tên_user_mới>@<IP_VPS>
```

Di chuyển hoặc clone lại repo bằng user non-root (nếu muốn sở hữu file):

```bash
sudo mv /root/setup-vps ~/
cd ~/setup-vps
sudo chown -R $(whoami):$(whoami) .
```

Hoặc clone mới nếu thích:

```bash
git clone https://github.com/reishou/chomusuke-vps-bash.git ~/setup-vps
cd ~/setup-vps
```

## Đổi port SSH

Chạy script:

```bash
./change-ssh-port.sh
```

Script sẽ hỏi port mới (khuyến nghị chọn port cao như 2222, 4321, 60022... để giảm bot tấn công). Nó sẽ sửa `/etc/ssh/sshd_config` và restart dịch vụ SSH.

{% hint style="info" %}
**Lưu ý quan trọng**: Không đóng terminal hiện tại. Mở tab/terminal mới trên local để test kết nối:

```
ssh -p <port_mới> <tên_user_mới>@<IP_VPS>
```
{% endhint %}

Nếu kết nối thành công, bạn có thể đóng session cũ.

## Tạo và cấu hình SSH config trên local (dùng setup-local-ssh.sh)

Chạy script này **trên máy local** (không phải trên VPS):

```bash
cd <thư-mục-repo-trên-local>   # hoặc clone repo về local nếu chưa có
git clone https://github.com/reishou/chomusuke-vps-bash.git
cd chomusuke-vps-bash
./setup-local-ssh.sh
```

Script sẽ:

* Generate SSH key pair mới trên local nếu chưa có (thường lưu tại `~/.ssh/id_rsa` và `~/.ssh/id_rsa.pub`).
* Hỏi thông tin: IP VPS, user non-root, port SSH mới.
* Tự động copy public key lên VPS (vào \~/.ssh/authorized\_keys của user non-root).
* Tạo hoặc cập nhật file \~/.ssh/config trên local với entry mẫu:

```bash
Host myserver
    HostName <IP_VPS>
    User <tên_user_mới>
    Port <port_mới>
    IdentityFile ~/.ssh/id_rsa
```

Test kết nối đơn giản:

```bash
ssh myserver
```

Từ giờ bạn connect rất tiện lợi, không cần chỉ định port hay user mỗi lần.

## Chạy script chính setup-vps.sh

Đăng nhập lại vào VPS bằng user non-root:

```bash
ssh myserver
```

Vào thư mục repo:

```bash
cd ~/setup-vps
```

Chạy script chính:

```bash
./setup-vps.sh
```

Hoặc quiet mode (ít output, không in footer trang trí):

```bash
./setup-vps.sh --quiet
```

Script sẽ chạy lần lượt các bước trong `./scripts/`:

* `setup-basic-security.sh` (bảo mật cơ bản: fail2ban, ufw, sysctl hardening...).
* `setup-basic-server.sh` (cài gói cơ bản, timezone, unattended-upgrades...).
* `setup-common-stack.sh` (stack phổ biến: nginx, PHP, PostgresQL/Redis... tùy config).

Thời gian chạy khoảng 5–15 phút tùy tốc độ VPS.

## Kiểm tra sau khi hoàn tất

Sau khi script báo hoàn tất:

* Kiểm tra dịch vụ: `systemctl status nginx`, `php -v`, postgres --version...
* Firewall: `sudo ufw status` (nên chỉ mở port SSH mới + 80/443 nếu có web).
* Fail2ban (nếu có): `sudo fail2ban-client status sshd`.
* Reboot VPS và test kết nối: `sudo reboot` rồi chờ 1–2 phút, sau đó `ssh myserver`.

Nếu mọi thứ ổn, server đã sẵn sàng sử dụng.

## Kết luận

Quy trình trên giúp VPS an toàn hơn nhiều so với mặc định: không còn root login qua SSH, chỉ key-based authentication, port SSH không chuẩn, user non-root, và các lớp bảo mật cơ bản được áp dụng tự động qua script.

Nếu muốn tùy chỉnh:

* Đọc kỹ từng file trong `./scripts/` để hiểu và chỉnh sửa theo nhu cầu.
* Cập nhật repo định kỳ: `git pull` trong thư mục setup-vps.
* Luôn test script trên VPS staging/test trước khi áp dụng cho production.

Chúc bạn setup thành công và có server ổn định! Nếu gặp lỗi, kiểm tra output script hoặc log hệ thống trong `/var/log/`.
