---
icon: rocket
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/local-automation/one-click-deploy-script-for-astro
---

# Script triển khai Astro tự động bằng Bash

## Yêu cầu trước khi chạy

* VPS Ubuntu/Debian (22.04 hoặc 24.04 khuyến nghị)
* Đã setup user non-root + SSH key (từ các script trước trong repo)
*   Đã clone repo về VPS:<br>

    ```bash
    git clone https://github.com/reishou/chomusuke-vps-bash.git
    cd chomusuke-vps-bash
    ```


* Quyền sudo (script cần install gói nếu thiếu)
* Domain đã trỏ về IP VPS (A record cho HTTP, A/AAAA cho HTTPS nếu dùng Certbot)

## Chuẩn bị SSH key cho GitHub (nếu repo private hoặc cần push/pull sau này)

Script `deploy-astro.sh` sử dụng `git clone` để lấy mã nguồn từ GitHub. Nếu repo của bạn là **private** hoặc bạn muốn push code/update từ VPS sau này, cần có SSH key đã add vào GitHub.

**Trước khi chạy deploy-astro.sh, hãy chạy script generate SSH key nếu chưa có**:

```bash
chmod x+ scripts/*.sh
./scripts/setup-vps-ssh.sh
```

Script sẽ:

* Hỏi tên key bạn muốn (default: `id_ed25519`, bạn có thể nhập tùy ý như `github_vps_ed25519`)
* Kiểm tra key cũ có tồn tại không → hỏi overwrite nếu có
* Hỏi có đặt passphrase không (khuyến nghị Yes cho bảo mật)
* Tạo key pair ed25519 (an toàn và nhanh)
* Hiển thị **public key** dạng text để copy

Copy public key được hiển thị (dạng `ssh-ed25519 AAAAC3Nza... user@hostname`):

* Vào GitHub → Settings → SSH and GPG keys → New SSH key
* Paste public key vào ô "Key"
* Đặt Title ví dụ: `VPS-$(hostname) - github_vps_ed25519 - 2025`
* Nhấn Add SSH key

Test kết nối GitHub:

```bash
ssh -T git@github.com
```

Nếu thấy thông báo: `Hi username! You've successfully authenticated...` → thành công.

**Lưu ý**:

* Nếu bạn đã có SSH key và add vào GitHub rồi → có thể skip bước này.
* Script `deploy-astro.sh` sẽ tự clone bằng HTTPS nếu repo public (không cần key), nhưng dùng SSH sẽ tiện hơn cho private repo và các thao tác sau.

## Các tính năng chính của script

* Kiểm tra và cài Nginx nếu chưa có (hỏi xác nhận)
* Nhập Git URL repo Astro → tự động extract tên folder
* Hỏi tên folder clone (default từ repo name)
* Xử lý folder đã tồn tại (hỏi overwrite)
* Clone repo → npm install → npm run build
* Hỏi domain và root path (default /var/www/\<folder>/dist)
* Đồng bộ build output (dist/) vào /var/www/ với quyền www-data
* Hỏi SSL: ưu tiên file key/cert có sẵn (Cloudflare, ZeroSSL,...) → nếu không thì hỏi dùng Certbot auto
* Tạo Nginx config từ template `./config/astro.conf.example`
* Thay thế placeholder (domain, root, SSL paths)
* Enable site, test config và reload Nginx
* In summary: URL HTTP/HTTPS, thư mục site, root path, config file

## Cách chạy script

Vào thư mục repo:

```bash
cd ~/chomusuke-vps-bash
```

Cấp quyền thực thi (nếu chưa):

```bash
chmod +x scripts/apps/deploy-astro.sh
```

Chạy script (không sử dụng sudo):

```bash
./scripts/apps/deploy-astro.sh
```

Script sẽ hỏi từng bước một cách rõ ràng (sử dụng hàm `ask_confirm` từ `utils.sh` để default Enter là Yes/No hợp lý).

## Flow tương tác chi tiết

Script chạy theo wizard-style:

1. **Nginx**: Nếu chưa cài → hỏi "Do you want to install Nginx now? \[Y/n]"
2. **Git URL**: Nhập URL repo (https hoặc git@)
3. **Folder name**: Default từ tên repo, hỏi nếu muốn đổi
4. **Overwrite folder**: Nếu folder tồn tại → hỏi xóa không
5. **Build Astro**: Tự động npm install & npm run build
6. **Domain**: Nhập domain (ví dụ example.com)
7. **Root path**: Default dist/, hỏi nếu muốn đổi
8. **SSL**:
   * Hỏi "Do you have existing SSL key and cert files? \[Y/n]"
     * Nếu Yes → hỏi path origin.key và origin.crt
     * Validate file tồn tại + format file
   * Nếu No → hỏi "Do you want to use Certbot? \[Y/n]"
     * Nếu Yes → tự cài Certbot nếu thiếu → chạy certbot --nginx
9. **Nginx config**: Copy từ template → thay placeholder → enable site → nginx -t → reload

Sau khi hoàn tất, script in summary với URL truy cập.

## Template Nginx (./config/astro.conf.example)

Template được thiết kế tối ưu cho Astro static site:

* Hỗ trợ IPv4 + IPv6
* Gzip compression, cache assets
* try\_files cho SPA routing (/ → /index.html)
* Security headers cơ bản
* Phần SSL được comment sẵn, script sẽ uncomment nếu có key/cert

Bạn có thể tùy chỉnh thêm trong template (ví dụ thêm brotli, proxy headers nếu dùng Cloudflare).

## Lợi ích khi dùng script

* Tiết kiệm thời gian: Từ clone đến live chỉ vài phút thay vì thủ công 15-30 phút
* Giảm lỗi: Validation input, default hợp lý, test nginx -t trước reload
* Linh hoạt SSL: Hỗ trợ file có sẵn (Cloudflare Origin CA) hoặc Certbot Let's Encrypt
* Nhất quán: Dùng chung utils.sh, màu log, prompt tiếng Anh
* Dễ mở rộng: Sau này có thể thêm auto git pull + rebuild qua cron

## Lưu ý & troubleshooting

* Script chạy với quyền sudo (cần để install gói và config /etc/nginx)
* Domain phải trỏ đúng IP VPS trước khi chạy Certbot
* Nếu Certbot fail → kiểm tra DNS (dig example.com) và firewall (ufw allow 80,443)
* Build Astro yêu cầu Node.js ≥ 18 (script giả định đã có từ setup trước)
* Nếu gặp lỗi npm → chạy `nvm use` hoặc `apt install nodejs npm` thủ công trước

## Kết luận

`deploy-astro.sh` là một phần trong bộ tool tự động hóa VPS của mình (chomusuke-vps-bash). Bạn có thể fork repo, customize template hoặc thêm hỗ trợ cho các framework khác (Laravel, Next.js) dựa trên flow tương tự.

Nếu bạn deploy Astro thường xuyên, script này sẽ giúp tiết kiệm rất nhiều thời gian. Chúc bạn có những site Astro nhanh và đẹp!

Repo: [https://github.com/reishou/chomusuke-vps-bash](https://github.com/reishou/chomusuke-vps-bash)

File script: [https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-astro.sh](https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-astro.sh)
