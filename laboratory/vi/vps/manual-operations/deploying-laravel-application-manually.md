---
icon: laravel
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/deploying-laravel-application-manually
---

# Deploy Thủ Công Ứng Dụng Laravel

**Laravel** là một framework PHP full-stack mạnh mẽ, được sử dụng rộng rãi để xây dựng các ứng dụng web hiện đại. Khác với static site như Astro (chỉ build một lần và serve file tĩnh), Laravel cần runtime PHP-FPM liên tục, kết nối database, quản lý queue worker, và quyền truy cập file rất chặt chẽ. Đây là phần deploy phức tạp nhất trong series, nhưng nếu làm đúng, bạn sẽ có một ứng dụng production-ready trên VPS Ubuntu.

**Giả sử**:

* Project Laravel phiên bản 11 hoặc 12.
* Repo GitHub đã có (ví dụ: git@github.com:reishou/chomusuke-demo-lara.git).
* Server đã có Nginx, PHP 8.4 + FPM, Composer, Node.js, Git (với SSH key deploy từ Common Stack).

**Lý do chọn PostgreSQL thay vì MySQL/MariaDB**

**PostgreSQL** (Postgres) có nhiều ưu điểm vượt trội so với MySQL trong các ứng dụng hiện đại: hỗ trợ JSONB mạnh mẽ, full-text search tốt, concurrency cao hơn, type safety nghiêm ngặt, và ít "quirks" hơn MySQL. Laravel từ phiên bản mới khuyến khích dùng Postgres với driver pgsql, và nó là lựa chọn lý tưởng nếu app của bạn cần query phức tạp, scale tốt hoặc bạn thích SQL chuẩn.

{% hint style="info" %}
Nếu project của bạn dùng MySQL, chỉ cần thay DB\_CONNECTION=mysql và cài mariadb thay vì postgres.
{% endhint %}

## **Cài đặt thêm PHP extensions cần thiết cho Postgres + Redis**

Common Stack trước đó đã cài một số extension cơ bản, nhưng để kết nối Postgres và Redis hiệu quả, cần bổ sung:

```bash
sudo apt update
sudo apt install -y php8.4-pgsql php8.4-redis
```

* `php8.4-pgsql`: Driver để Laravel kết nối PostgreSQL.
* `php8.4-redis`: Extension phpredis (nhanh và được Laravel khuyến nghị hơn predis package).

Kiểm tra extensions đã cài

```bash
php -m | grep -E 'pgsql|redis'
```

Nếu thấy `pgsql` và `redis` trong danh sách → OK. Restart PHP-FPM để áp dụng:

```bash
sudo systemctl restart php8.4-fpm
```

## **Chuẩn bị thư mục deploy và quyền cơ bản**

Tạo thư mục cho project và set quyền đúng (để tránh lỗi 502/500 sau này):

```bash
sudo mkdir -p /var/www/chomusuke-demo-lara
sudo chown -R vps-user:www-data /var/www/chomusuke-demo-lara
sudo chmod -R 755 /var/www/chomusuke-demo-lara
```

* `vps-user`: user bạn đang login (thay nếu khác).
* `www-data`: group mặc định của Nginx và PHP-FPM.

## **Clone repo và cài dependencies**

```bash
cd /var/www/chomusuke-demo-lara
git clone git@github.com:reishou/chomusuke-demo-lara.git .
```

Cài dependencies production:

```bash
composer install --optimize-autoloader --no-dev --prefer-dist
```

* `--no-dev`: Bỏ qua package trong `require-dev` (như PHPUnit, debug tools) → vendor folder nhỏ hơn, an toàn hơn, không cần ở production.
* `--prefer-dist`: Tải package dưới dạng file zip/tar đã build sẵn thay vì clone git → nhanh hơn, nhẹ hơn, không chứa lịch sử git thừa.
* `--optimize-autoloader` (hoặc `-o`): Tạo autoloader kiểu classmap tĩnh (nhanh hơn PSR-4 động) → ứng dụng load class nhanh hơn đáng kể.

Nếu project dùng Vite cho frontend:

```bash
pnpm install
pnpm run build
```

## **Cài đặt và config PostgreSQL**

Cài PostgreSQL (nếu chưa có)

```bash
sudo apt install -y postgresql postgresql-contrib
```

Khởi động và enable:

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Kiểm tra

```bash
sudo systemctl status postgresql
```

Tạo database và user

```bash
sudo -u postgres psql
```

Trong psql shell

```sql
CREATE DATABASE lara;
CREATE USER lara_user WITH ENCRYPTED PASSWORD 'Abcd@1234';
GRANT ALL PRIVILEGES ON DATABASE lara TO lara_user;
\q
```

## **Config file .env production**

Copy file mẫu

```bash
cp .env.example .env
```

Chỉnh sửa `.env` (dùng `nano .env`)

```properties
APP_ENV=production
APP_DEBUG=false
APP_KEY=  # Chạy php artisan key:generate để tạo

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=lara
DB_USERNAME=lara_user
DB_PASSWORD=Abcd@1234

CACHE_STORE=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null  # Nếu set password thì điền vào
REDIS_PORT=6379
```

Chạy các lệnh artisan cần thiết:

```bash
php artisan key:generate
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

## **Cài đặt Redis (server + config)**

Cài Redis:

```bash
sudo apt install -y redis-server
```

Kiểm tra

```bash
redis-cli ping  # Nên trả về PONG
```

Config cơ bản (production nên bind chỉ localhost và thêm password nếu muốn):

```bash
sudo nano /etc/redis/redis.conf
```

Tìm và chỉnh:

```bash
bind 127.0.0.1 ::1
requirepass your_redis_password  # Optional nhưng recommend
```

Restart Redis:

```bash
sudo systemctl restart redis-server
```

## **Chạy migration và seed (nếu có)**

```bash
php artisan migrate --seed
```

> Lưu ý, vì `composer` đã install với tham số `--no-dev` nên trong seed không dùng các chức năng ở require-dev ví dụ như fake(), nếu không sẽ phát sinh lỗi<br>
>
> ```bash
> In UserFactory.php line 27:
>
>   Call to undefined function Database\Factories\fake()
> ```

Nếu lỗi kết nối DB → kiểm tra `.env` và password, hoặc xem log:

```bash
tail -f /var/log/postgresql/postgresql-*.log
```

## **Permissions – phần quan trọng nhất để tránh lỗi 502/500**

Lỗi 502 Bad Gateway thường do PHP-FPM không ghi được vào storage/bootstrap/cache hoặc socket permission sai.

Chạy lệnh sau:

```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

Nếu dùng Redis cho cache/session/queue, storage sẽ ít bị write hơn, nhưng vẫn cần quyền trên.

Kiểm tra log nếu lỗi:

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/php8.4-fpm.log
```

## **Cấu hình PHP-FPM pool**

Kiểm tra file pool (thường là default):

```bash
sudo nano /etc/php/8.4/fpm/pool.d/www.conf
```

Đảm bảo các dòng sau:

```properties
user = www-data
group = www-data
listen = /run/php/php8.4-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
pm = dynamic  # Hoặc ondemand cho tiết kiệm RAM
```

Restart:

```bash
sudo systemctl restart php8.4-fpm
```

## **Cấu hình Nginx server block cho Laravel**

```bash
sudo nano /etc/nginx/sites-available/chomusuke-demo-lara.conf
```

Paste nội dung:

```nginx
server {
    listen 80;
    server_name lara.chomusuke.site;  # Thay bằng domain, hoặc bỏ để test bằng IP

    root /var/www/chomusuke-demo-lara/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    access_log /var/log/nginx/chomusuke-demo-lara.access.log;
    error_log /var/log/nginx/chomusuke-demo-lara.error.log;
}
```

Lưu file → enable site

```bash
sudo ln -s /etc/nginx/sites-available/chomusuke-demo-lara.conf /etc/nginx/sites-enabled/
```

Khi syntax config kiểm tra thành công

```bash
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

thì mới reload nginx

```bash
sudo systemctl reload nginx
```

## **Setup queue worker với Supervisor (dùng Redis)**

Tạo file config:

```bash
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Paste:

```properties
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/chomusuke-demo-lara/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/chomusuke-demo-lara/storage/logs/worker.log
```

Reload supervisor:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart laravel-worker:*
```

Kiểm tra:

```bash
sudo supervisorctl status
```

## **Kiểm tra và debug lỗi phổ biến**

Truy cập [http://your-vps-ip](http://your-vps-ip/?referrer=grok.com) hoặc domain → thấy trang Laravel.

* **Lỗi 502 Bad Gateway**:
  * Kiểm tra `sudo systemctl status php8.4-fpm`
  * Socket permission: `ls -l /run/php/php8.4-fpm.sock` → phải là `srw-rw---- www-data www-data`
  * Extensions thiếu: `php -m` kiểm tra pgsql, redis
* **Lỗi 500**:
  * storage không writable → chạy lại chown/chmod
  * `.env` sai hoặc chưa cache → `php artisan config:clear`, `php artisan config:cache`
  * Queue không chạy → `supervisorctl status`, kiểm tra log worker

Test nhanh:

```bash
php artisan tinker   # gõ exit để thoát
curl http://localhost
```

## **Update code sau này**

```bash
cd /var/www/chomusuke-demo-lara
git pull origin main
composer install --no-dev --optimize-autoloader --prefer-dist
pnpm install && pnpm run build  # Nếu có frontend thay đổi
php artisan migrate --force   # Nếu có migration mới
php artisan queue:restart
sudo supervisorctl restart laravel-worker:*
sudo systemctl reload nginx
```

Vậy là xong! Ứng dụng Laravel của bạn đã live trên VPS thủ công với PostgreSQL và Redis cho queue/cache.
