---
icon: rocket
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/deploying-a-simple-static-site-manually-astro
---

# Deploy Thủ Công Trang Static Đơn Giản (Astro)

**Astro** là framework static-first cực kỳ phổ biến hiện nay, build ra HTML/CSS/JS tĩnh thuần túy, tốc độ tải nhanh, SEO tốt, và không cần runtime server-side liên tục.

Đây là ví dụ deploy đơn giản nhất trong series: chỉ cần clone code → install dependencies → build → serve bằng Nginx. Không cần process manager (pm2/supervisor), queue, database hay gì phức tạp.

Thời gian deploy lần đầu: khoảng 5-10 phút. Lý tưởng cho blog, landing page, portfolio, docs.

**Giả sử**:

* Bạn đã có repo Astro trên GitHub (ví dụ: `git@github.com:reishou/chomusuke-home.git`).
* Server Ubuntu đã có **Nginx**, **Node.js LTS**, **Git** (với SSH key deploy setup từ phần Common Stack).
* Project Astro dùng adapter static mặc định (không SSR/hybrid cần Node runtime).

## Chuẩn bị thư mục deploy

Tạo thư mục cho project và set quyền đúng (để Nginx đọc được file).

```bash
sudo mkdir -p /var/www/chomusuke-home
sudo chown -R vps-user:www-data /var/www/chomusuke-home
sudo chmod -R 755 /var/www/chomusuke-home
```

* `vps-user`: user bạn đang login (thay nếu khác).
* `www-data`: group mặc định của Nginx/PHP-FPM.

## Clone repo từ GitHub

```bash
cd /var/www/chomusuke-home

# Clone lần đầu (dùng SSH URL từ repo GitHub)
git clone git@github.com:reishou/chomusuke-home.git .

# Nếu đã clone rồi, chỉ cần update:
# git pull origin main
```

* Dấu `.` để clone thẳng vào folder hiện tại (không tạo subfolder thừa).
* Nếu repo dùng branch khác main, checkout branch đó: `git checkout production` (tùy project).

## Cài dependencies và build Astro

```bash
# Cài dependencies (dùng npm, hoặc pnpm/yarn nếu project của bạn dùng)
pnpm install

# Build static site → tạo folder dist/
pnpm run build
```

Sau lệnh này, folder `/var/www/chomusuke-home/dist` sẽ chứa toàn bộ file tĩnh (HTML, CSS, JS, images...).\
Đây là thứ Nginx sẽ serve.

## Cấu hình Nginx server block

Tạo file config riêng cho site.

```bash
sudo nano /etc/nginx/sites-available/chomusuke-home.conf
```

Paste nội dung cơ bản sau (thay domain nếu có, hoặc dùng IP để test)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name chomusuke.site; # Thay bằng domain thật, hoặc bỏ để test bằng IP

    root /var/www/chomusuke-home/dist;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;  # Serve file tĩnh trực tiếp, fallback 404 nếu không tìm thấy
    }

    # Tùy chọn: Log để debug
    access_log /var/log/nginx/chomusuke-home.access.log;
    error_log /var/log/nginx/chomusuke-home.error.log;
}
```

* `root`: trỏ thẳng đến folder dist sau build.
* `index`: file mặc định khi truy cập thư mục.
* `try_files`: tìm file chính xác, nếu không có thì trả 404 (đơn giản, phù hợp static site thuần).

Lưu file → enable site

```bash
sudo ln -s /etc/nginx/sites-available/chomusuke-home.conf /etc/nginx/sites-enabled/
```

Khi syntax config kiểm tra thành công&#x20;

```bash
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

thì mới reload nginx

```bash
sudo systemctl reload nginx
```

## Trỏ domain về server (DNS Setup)

Để truy cập site qua domain thay vì IP:

* Lấy IP VPS: `curl ifconfig.me` hoặc xem trong dashboard provider (Vultr, DigitalOcean...).
* Vào panel quản lý domain (Cloudflare, Namecheap, Godaddy...).
* Thêm/chỉnh A record:
  * Name/Host: `@` (cho domain gốc, ví dụ `chomusuke-home.com`) và `www` (nếu muốn [www.chomusuke-home.com](http://www.chomusuke-home.com/?referrer=grok.com)).
  * Value/Points to: IP VPS của bạn.
  * TTL: 300 (5 phút) hoặc mặc định.
* Chờ propagation (thường 5-30 phút, max 48h): dùng tool dnschecker.org hoặc lệnh `dig chomusuke-home.com` để check.
* Nếu dùng Cloudflare: set proxy (orange cloud) off ban đầu để debug dễ.

Test nhanh bằng IP: [http://your-vps-ip](http://your-vps-ip/?referrer=grok.com) → thấy site Astro.

## Set quyền file cuối cùng

```bash
sudo chown -R www-data:www-data /var/www/chomusuke-home/dist
sudo chmod -R 755 /var/www/chomusuke-home/dist
sudo find /var/www/chomusuke-home/dist -type f -exec chmod 644 {} \;
```

* Nginx chạy dưới user www-data → cần quyền đọc file.

## Kiểm tra và truy cập

* Mở browser: [http://your-vps-ip](http://your-vps-ip/?referrer=grok.com) hoặc [http://domain.com](http://domain.com/?referrer=grok.com) (sau khi DNS ok).
*   Kiểm tra log nếu lỗi:<br>

    ```bash
    sudo tail -f /var/log/nginx/chomusuke-home.error.log
    sudo tail -f /var/log/nginx/chomusuke-home.access.log
    ```


* Nếu thấy trang Astro load đẹp → thành công!

## Update code sau này (quy trình deploy lại)

Mỗi khi có thay đổi code

```bash
cd /var/www/chomusuke-home
git pull origin main   # Update code
npm install            # Nếu dependencies thay đổi
npm run build          # Build lại dist/
sudo systemctl reload nginx   # Reload config nếu cần
```

Xong! Site static Astro của bạn đã live trên VPS thủ công.
