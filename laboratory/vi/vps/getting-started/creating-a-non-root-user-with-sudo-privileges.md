---
icon: user-gear
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/readme/creating-a-non-root-user-with-sudo-privileges
---

# Tạo User Thường Có Quyền Sudo

Sau khi bạn đã SSH thành công lần đầu vào VPS với tài khoản **root**, bước bảo mật quan trọng tiếp theo là **tạo một user thường có quyền sudo**.

Không nên sử dụng root để làm việc hàng ngày vì rủi ro bảo mật rất cao (nếu bị tấn công, kẻ xấu sẽ có toàn quyền). Thay vào đó, tạo user riêng (ví dụ: `vps-user`), cấp quyền sudo để chạy lệnh cần quyền root khi cần, và sau đó tắt đăng nhập root trực tiếp.

Dưới đây là hướng dẫn chi tiết từng bước trên **Ubuntu/Debian**.

## Tạo user mới

```bash
adduser vps-user
```

* Lệnh `adduser` là cách thân thiện, tương tác để tạo user mới (khác với `useradd` cơ bản hơn).
* Sau khi chạy, hệ thống sẽ hỏi bạn:
  * Mật khẩu cho user mới (nhập 2 lần).
  * Một số thông tin cá nhân (Full Name, Room Number, v.v.) → có thể bỏ qua bằng cách Enter liên tục.
* User sẽ được tạo, thư mục home `/home/vps-user` tự động được tạo.

Dưới đây là ví dụ output khi chạy lệnh:

```bash
root@ubuntu:~$ adduser vps-user
[sudo] password for root:
info: Adding user `vps-user' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `vps-user' (1000) ...
info: Adding new user `vps-user' (1000) with group `vps-user (1000)' ...
info: Creating home directory `/home/vps-user' ...
info: Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for vps-user
Enter the new value, or press ENTER for the default
	Full Name []:
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n]
info: Adding new user `vps-user' to supplemental / extra groups `users' ...
info: Adding user `vps-user' to group `users' ...
root@ubuntu:~$ 
```

## Cấp quyền sudo cho user mới

```bash
usermod -aG sudo vps-user
```

* `-aG`: Append (thêm) vào group, không xóa các group cũ.
* `sudo`: Tên group mặc định trên Ubuntu/Debian cho phép chạy lệnh với quyền root qua `sudo`.
* Sau lệnh này, user `vps-user` có thể chạy lệnh như `sudo apt update` bằng cách nhập mật khẩu của chính mình (không phải root).

## Tạo SSH key trên máy local

#### Kiểm tra xem đã có SSH key chưa

Mở **Terminal** trên máy local (**không phải trên server)**

```bash
ls -al ~/.ssh
```

Nếu bạn thấy các file như `id_ed25519` và `id_ed25519.pub` (hoặc `id_rsa`, `id_ecdsa`, v.v.) thì bạn **đã có key**. Có thể bỏ qua bước tạo mới và đi thẳng đến phần copy key.

Nếu thư mục `~/.ssh` chưa tồn tại hoặc không có file key nào → tiếp tục tạo mới.

#### Tạo SSH key mới (khuyến nghị dùng Ed25519 – hiện đại, an toàn nhất)

Chạy lệnh sau trong Terminal (trên máy local):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

* `-t ed25519`: Loại key Ed25519 (nhanh, an toàn hơn RSA cũ, được khuyến nghị năm 2025+).
* `-C "your_email@example.com"`: Comment để nhận diện key (thường là email của bạn, giúp dễ quản lý nhiều key).

Quá trình sẽ hỏi:

1. **Enter file in which to save the key** (`/home/yourname/.ssh/id_ed25519` hoặc `/Users/yourname/.ssh/id_ed25519`): \
   → Nhấn **Enter** để dùng mặc định (khuyến nghị).
2. **Enter passphrase** (mật khẩu bảo vệ key):
   * Nhập passphrase mạnh (khuyến nghị) hoặc **Enter** để bỏ qua (không passphrase – tiện nhưng kém an toàn hơn một chút).
   * Nếu nhập passphrase, bạn sẽ phải gõ mỗi lần SSH (có thể dùng ssh-agent để cache).

**Output thành công ví dụ**:

```bash
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/user/.ssh/id_ed25519
Your public key has been saved in /home/user/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx your_email@example.com
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|        .        |
|       + o       |
|      . B o      |
|     . S B .     |
|    . . O + .    |
|   . . + = o     |
|    . . o + .    |
|     . . .E.     |
+----[SHA256]-----+
```

Bây giờ bạn đã có 2 file quan trọng:

* `~/.ssh/id_ed25519` → **Private key** (bí mật, **không bao giờ chia sẻ!**)
* `~/.ssh/id_ed25519.pub` → **Public key** (đây là phần bạn sẽ copy lên VPS)

## Copy SSH key cho user mới (khuyến nghị dùng key thay vì password)

Cách tốt nhất và an toàn nhất là dùng SSH key.

#### **Cách 1 – Tự động dùng ssh-copy-id** (khuyến nghị):

Từ máy local của bạn (**không phải trên server**), chạy:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub vps-user@123.45.67.89
```

* Bạn sẽ được hỏi mật khẩu của `vps-user` (lần đầu).
* Public key từ máy local sẽ tự động copy vào file `~/.ssh/authorized_keys` của user `vps-user`.

Ví dụ minh họa lệnh ssh-copy-id:

{% code overflow="wrap" %}
```bash
~$ ssh-copy-id -i ~/.ssh/id_ed25519.pub vps-user@123.45.67.89
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/reishou/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
vps-user@123.45.67.89's password:

Number of key(s) added:        1

Now try logging into the machine, with: "ssh -i /Users/reishou/.ssh/id_ed25519 'vps-user@123.45.67.89'"
and check to make sure that only the key(s) you wanted were added.
```
{% endcode %}

#### **Cách 2 – Manual (nếu ssh-copy-id không dùng được)**:

* Trên máy local: `cat ~/.ssh/id_ed25519.pub` (hoặc `id_rsa.pub`) → copy nội dung.
* Trên server (đăng nhập root):

```bash
mkdir -p /home/vps-user/.ssh
chown vps-user:vps-user /home/vps-user/.ssh
chmod 700 /home/vps-user/.ssh
nano /home/vps-user/.ssh/authorized_keys
```

Paste public key vào, lưu lại.

```bash
chown vps-user:vps-user /home/vps-user/.ssh/authorized_keys
chmod 600 /home/vps-user/.ssh/authorized_keys
```

## Test đăng nhập bằng user mới

Từ máy local, thử đăng nhập:

```bash
ssh vps-user@123.45.67.89
```

Nếu dùng key → sẽ vào thẳng mà không hỏi mật khẩu.

Nếu dùng password → nhập mật khẩu của `vps-user`.

Sau khi login thành công, test quyền sudo:

```bash
sudo whoami
```

→ Nhập mật khẩu của `vps-user` → Output phải là `root`.

Ví dụ terminal sau khi login và dùng sudo thành công:

```
Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-90-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jan  9 05:33:00 PM UTC 2026

  System load:  0.0               Processes:             122
  Usage of /:   37.3% of 9.76GB   Users logged in:       0
  Memory usage: 16%               IPv4 address for eth0: 123.45.67.89
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Fri Jan  9 16:20:05 2026 from 123.45.67.89
vps-user@123.45.67.89:~$ sudo whoami
[sudo] password for vps-user:
root
vps-user@123.45.67.89:~$
```

