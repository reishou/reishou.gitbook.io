---
icon: node-js
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/local-automation/one-click-deploy-script-for-nestjs
---

# Script triển khai NextJS tự động bằng Bash

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
* **SSH key đã add vào GitHub** nếu repo private (chạy `setup-vps-ssh.sh` trước nếu chưa có)

**Lưu ý quan trọng trước khi chạy**:

* Script sẽ clone repo Next.js vào thư mục bạn chọn (ví dụ `/home/col-user/my-next-app`).
* Sau đó **rsync build output** (thư mục `out/` hoặc `.next/`) vào `/var/www/my-next-app` để Nginx đọc (best practice: source code ở home user writable, web root ở `/var/www` read-only).
* Đảm bảo chạy script từ **root repo** (`chomusuke-vps-bash/`), không cd vào thư mục project trước.

## Các tính năng chính của script

* Kiểm tra và cài tự động các công cụ cần thiết: nginx, pm2, node, pnpm, postgresql (psql)
* Hỏi tạo DB PostgreSQL + user (tùy chọn, không bắt buộc)
* Hỏi cấu hình .env:&#x20;
  * POSTGRES\_URL (ghép từ DB input, default `postgresql://next_user:Abcd@1234@127.0.0.1:5432/next`),&#x20;
  * AUTH\_SECRET (tự generate 32 ký tự random),&#x20;
  * AUTH\_URL
* Clone repo → pnpm install → pnpm run build
* Rsync build output vào `/var/www`, `chown www-data`
* Hỏi domain (default: next.chomusuke.site)
* Hỏi SSL: ưu tiên file key/cert có sẵn (default `/etc/ssl/cloudflare/chomusuke.site.key` và `.crt`) → nếu không thì hỏi Certbot auto
* Tạo Nginx config từ template `./config/nginx/next.conf.example` (copy thành `/etc/nginx/sites-available/{folder_name}.conf`)
* Start app với PM2 (dùng `ecosystem.config.js` có sẵn trong repo)
* In `pm2 status` để kiểm tra app chạy

## Cách chạy script

Vào thư mục repo:

```bash
cd ~/chomusuke-vps-bash
```

Cấp quyền thực thi:

```bash
chmod +x scripts/apps/deploy-next.sh
```

Chạy script (không cần sudo toàn bộ, script tự dùng sudo khi cần):

```bash
./scripts/apps/deploy-next.sh
```

Script sẽ hỏi từng bước một cách rõ ràng (sử dụng `ask_confirm` từ `utils.sh` để default Enter hợp lý).

## Flow tương tác chi tiết

1. **Prerequisites**: Kiểm tra nginx, pm2, node, pnpm, psql → hỏi cài nếu thiếu.
2. **PostgreSQL DB** (tùy chọn): Hỏi có tạo DB + user không → nhập tên DB/user/pass nếu yes.
3. **Git repo**: Nhập URL (hỗ trợ SSH & HTTPS), verify bằng `git ls-remote`.
4. **Folder name**: Default từ repo name, hỏi nếu muốn đổi.
5. **.env config**: Hỏi POSTGRES\_URL (ghép từ input), generate AUTH\_SECRET, AUTH\_URL.
6. **Build**: pnpm install + pnpm run build.
7. **Domain**: Nhập domain (default `next.chomusuke.site`).
8. **Root path & rsync**: Default out/, rsync build output vào `/var/www`, `chown www-data`.
9. **SSL**: Hỏi file key/cert có sẵn (default `chomusuke.site.key/.crt`) → nếu không thì hỏi Certbot.
10. **Nginx config**: Copy từ template, thay placeholder, apply.
11. **PM2 start**: Start app dùng `ecosystem.config.js` có sẵn, in pm2 status.
12. **Summary**: In URL HTTP/HTTPS, thư mục site, root path, config file.

#### Template chính (đã dùng trong script)

* **Nginx**: `./config/nginx/next.conf.example` (hỗ trợ static export hoặc server mode, log theo folder\_name)
* **PM2**: Dùng `ecosystem.config.js` có sẵn trong repo Next.js (script chỉ start và save)

#### Lợi ích khi dùng script

* Tiết kiệm thời gian: Từ clone đến live chỉ 5-10 phút thay vì thủ công 30-60 phút.
* Giảm lỗi: Validation input, default hợp lý, test `nginx -t` trước reload, fix permission tự động.
* Linh hoạt SSL: Hỗ trợ file có sẵn (Cloudflare Origin CA) hoặc Certbot Let's Encrypt.
* Nhất quán: Dùng chung `utils.sh`, màu log, prompt tiếng Anh.
* Dễ mở rộng: Có thể thêm auto migrate, seed, storage:link nếu dùng Prisma.

#### Lưu ý & troubleshooting

* Script chạy **không cần sudo toàn bộ** – chỉ dùng sudo cho apt, chown, systemctl, certbot.
* Domain phải trỏ đúng IP VPS trước khi chạy Certbot.
* Nếu lỗi "Permission denied" ở storage/logs: Script đã fix tự động sau rsync.
* Nếu build fail: Đảm bảo Node.js ≥ 18, pnpm installed.
* Nếu PM2 không chạy: Check log `pm2 logs` hoặc `ecosystem.config.js` có script start đúng không.
* Repo mẫu: [https://github.com/reishou/chomusuke-demo-next](https://github.com/reishou/chomusuke-demo-next) (dùng để test script).

#### Kết luận

`deploy-next.sh` là một phần quan trọng trong bộ tool tự động hóa VPS của mình (chomusuke-vps-bash). Bạn có thể fork repo, customize template hoặc mở rộng cho các framework khác dựa trên flow tương tự.

Nếu bạn deploy Next.js thường xuyên, script này sẽ giúp tiết kiệm rất nhiều thời gian và tránh lỗi thường gặp. Chúc bạn có những project Next.js ổn định và nhanh!

Repo: [https://github.com/reishou/chomusuke-vps-bash](https://github.com/reishou/chomusuke-vps-bash)&#x20;

File script: [https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-next.sh](https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-next.sh)&#x20;

Repo Next.js mẫu: [https://github.com/reishou/chomusuke-demo-next](https://github.com/reishou/chomusuke-demo-next)

Domain demo: [https://next.chomusuke.site](https://next.chomusuke.site)
