---
icon: laravel
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/local-automation/one-click-deploy-script-for-laravel
---

# Script triển khai Laravel tự động bằng Bash

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

* Script sẽ clone repo Laravel vào thư mục bạn chọn (ví dụ `/home/col-user/my-laravel-app`).
* Sau đó **rsync toàn bộ source code** vào `/var/www/my-laravel-app` để Nginx đọc (best practice: source code ở home user writable, web root ở `/var/www` read-only).
* Đảm bảo chạy script từ **root repo** (`chomusuke-vps-bash/`), không cd vào thư mục project trước.

## Các tính năng chính của script

* Kiểm tra và cài tự động các công cụ cần thiết: nginx, pnpm, php, composer, rsync, postgresql (psql), php extensions (bcmath, mbstring, pgsql, curl, gd, intl, xml, zip)
* Hỏi cài Redis (tùy chọn, không bắt buộc)
* Hỏi tạo DB PostgreSQL + user (tùy chọn, không bắt buộc)
* Hỏi cấu hình .env (DB connection, host/port, username/password, APP\_ENV=production, APP\_DEBUG=false, APP\_KEY generate)
* Cấu hình Redis trong .env nếu Redis đã cài
* Chạy `php artisan migrate` (tự động nếu tạo DB, hoặc hỏi nếu không)
* Chạy cache commands: config:cache, route:cache, view:cache (sau rsync để path đúng)
* Cấu hình php-fpm pool (user/group=www-data, socket, pm=dynamic)
* Hỏi domain và root path (default /var/www/\<folder>/public)
* Hỏi SSL: ưu tiên file key/cert có sẵn → nếu không thì hỏi Certbot auto
* Tạo Nginx config từ template `./config/nginx/laravel.conf.example`
* Cấu hình Supervisor cho queue workers từ template `./config/supervisor/laravel-worker.conf.example` (tùy chọn)
* In summary: URL, thư mục site, root path, config file

## Cách chạy script

Vào thư mục repo:

```bash
cd ~/chomusuke-vps-bash
```

Cấp quyền thực thi:

```bash
chmod +x scripts/apps/deploy-laravel.sh
```

Chạy script (không cần sudo toàn bộ, script tự dùng sudo khi cần):

```bash
./scripts/apps/deploy-laravel.sh
```

Script sẽ hỏi từng bước một cách rõ ràng (sử dụng `ask_confirm` từ `utils.sh` để default Enter hợp lý).

## Flow tương tác chi tiết

1. **Prerequisites**: Kiểm tra nginx, pnpm, php, composer, rsync, postgresql, php extensions → hỏi cài nếu thiếu.
2. **Redis** (tùy chọn): Hỏi có cài không nếu chưa có.
3. **PostgreSQL DB** (tùy chọn): Hỏi có tạo DB + user không → nhập tên DB/user/pass nếu yes.
4. **Git repo**: Nhập URL (hỗ trợ SSH & HTTPS), verify bằng `git ls-remote`.
5. **Folder name**: Default từ repo name, hỏi nếu muốn đổi.
6. **Composer install**: Cài dependencies.
7. **Vite build**: Install frontend deps + build assets (`pnpm install` & `pnpm run build`).
8. **.env config**: Hỏi DB, APP\_ENV=production, APP\_DEBUG=false, generate APP\_KEY (tùy chọn).
9. **Redis in .env**: Tự động apply nếu Redis installed.
10. **Domain**: Nhập domain.
11. **Root path & rsync**: Default `/public`, rsync full app vào `/var/www`, `chown www-data`.
12. **SSL**: Hỏi file key/cert có sẵn → nếu không thì hỏi Certbot.
13. **Nginx config**: Copy từ template, thay placeholder, apply.
14. **Supervisor**: Hỏi setup queue worker nếu supervisorctl có sẵn.
15. **Summary**: In URL HTTP/HTTPS, thư mục site, root path, config file.

## Template chính (đã dùng trong script)

* **Nginx**: `./config/nginx/laravel.conf.example` (location php-fpm, root /public, log theo folder\_name)
* **Supervisor**: `./config/supervisor/laravel-worker.conf.example` (queue:work với 4 processes, log `storage/logs/worker.log`)

## Lợi ích khi dùng script

* Tiết kiệm thời gian: Từ clone đến live chỉ 5-10 phút thay vì thủ công 30-60 phút.
* Giảm lỗi: Validation input, default hợp lý, test nginx -t trước reload, fix permission storage/logs/cache tự động.
* Linh hoạt SSL: Hỗ trợ file có sẵn (Cloudflare Origin CA) hoặc Certbot Let's Encrypt.
* Nhất quán: Dùng chung utils.sh, màu log, prompt tiếng Anh.
* Dễ mở rộng: Có thể thêm auto migrate, seed, storage:link, queue restart.

## Lưu ý & troubleshooting

* Script chạy **không cần sudo toàn bộ** – chỉ dùng sudo cho apt, chown, systemctl, certbot.
* Domain phải trỏ đúng IP VPS trước khi chạy Certbot.
* Nếu lỗi "Permission denied" ở storage/logs: Script đã fix tự động sau rsync.
* Nếu lỗi Vite build: Đảm bảo Node.js ≥ 18, pnpm installed.
* Nếu queue worker không chạy: Check log `/var/www/<folder>/storage/logs/worker.log` hoặc `sudo supervisorctl status`.
* Repo mẫu: [https://github.com/reishou/chomusuke-demo-lara](https://github.com/reishou/chomusuke-demo-lara) (dùng để test script).

## Kết luận

`deploy-laravel.sh` là một phần quan trọng trong bộ tool tự động hóa VPS của mình (`chomusuke-vps-bash`). Bạn có thể fork repo, customize template hoặc mở rộng cho các framework khác (Next.js, NestJS, v.v.) dựa trên flow tương tự.

Nếu bạn deploy Laravel thường xuyên, script này sẽ giúp tiết kiệm rất nhiều thời gian và tránh lỗi thường gặp. Chúc bạn có những project Laravel ổn định và nhanh!

Repo: [https://github.com/reishou/chomusuke-vps-bash](https://github.com/reishou/chomusuke-vps-bash)&#x20;

File script: [https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-laravel.sh](https://github.com/reishou/chomusuke-vps-bash/blob/main/scripts/apps/deploy-laravel.sh)&#x20;

Repo Laravel mẫu: [https://github.com/reishou/chomusuke-demo-lara](https://github.com/reishou/chomusuke-demo-lara)&#x20;

Domain mẫu: [https://lara.chomusuke.site/](https://lara.chomusuke.site/)
