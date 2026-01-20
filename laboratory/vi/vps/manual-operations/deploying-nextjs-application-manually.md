---
icon: node-js
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/deploying-nestjs-application-manually
---

# Deploy Thủ Công Ứng Dụng NextJS

**Next.js** là framework React full-stack mạnh mẽ, hỗ trợ hybrid rendering (SSG + ISR + SSR), App Router, Route Handlers (API backend), và UI đẹp sẵn (Tailwind, shadcn/ui...). Phần này deploy một ứng dụng Next.js hybrid: trang tĩnh (SSG) nhanh/SEO tốt + trang semi-dynamic (ISR) cho nội dung thay đổi (blog, products...).

Giả sử:

* Project Next.js 14/15, App Router.
* Repo GitHub: `git@github.com:reishou/chomusuke-demo-next.git`
* Sử dụng PostgreSQL (chung instance với Laravel) cho data backend.
* Redis (chung instance) cho caching (API responses, rate limiting...).
* Domain: `next.chomusuke.site` (chỉ dùng trong `server_name` của Nginx, không dùng để đặt tên folder/config).

## **Chuẩn bị thư mục deploy**

```bash
sudo mkdir -p /var/www/chomusuke-demo-next
sudo chown -R vps-user:www-data /var/www/chomusuke-demo-next
sudo chmod -R 755 /var/www/chomusuke-demo-next
```

* `.next/` folder sau build cần writable nếu dùng ISR (cache regeneration).

## **Clone repo**

```bash
cd /var/www/chomusuke-demo-next
git clone git@github.com:reishou/chomusuke-demo-next.git .
```

## **Cài đặt thêm package cần thiết (nếu chưa có)**

Cài client Redis cho Node.js (ioredis khuyến nghị nhanh hơn redis package):

```bash
pnpm add ioredis
```

Nếu project dùng PostgreSQL client (ví dụ drizzle-orm, prisma, pg):

```bash
pnpm add pg  # hoặc prisma, drizzle-orm tùy project
```

## **Config environment variables (.env)**

Copy file mẫu:

```bash
cp .env.example .env
```

Chỉnh sửa .env (nano .env):

```properties
NODE_ENV=production
PORT=3001  # Khác 3000 để tránh conflict nếu có app Node khác

# Redis (chung instance với Laravel)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=

# PostgreSQL (database riêng cho Next.js)
DATABASE_URL=postgresql://next_user:Abcd@1234@127.0.0.1:5432/next?schema=public

# NEXT_PUBLIC_ vars cho client-side
NEXT_PUBLIC_SITE_URL=https://next.chomusuke.site
NEXT_PUBLIC_API_URL=https://next.chomusuke.site/api
```

**Lưu ý Redis prefix** (để data độc lập với Laravel): Trong code Next.js (thường ở `lib/redis.ts` hoặc `utils/cache.ts`), tạo client với prefix:

```typescript
import Redis from 'ioredis';

export const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  password: process.env.REDIS_PASSWORD,
  keyPrefix: 'nextjs:',  // Prefix riêng cho Next.js (độc lập với 'laravel:' ở Laravel)
});
```

## **Cài dependencies và build**

```bash
pnpm install  # Khuyến nghị pnpm cho nhanh và tiết kiệm disk
pnpm build    # Tạo .next/ với static chunks + server bundles
```

## **Cài đặt và config PostgreSQL cho Next.js**

PostgreSQL đã cài từ phần Laravel (host 127.0.0.1:5432). Tạo database và user riêng cho Next.js:

```bash
sudo -u postgres psql
```

Trong psql shell:

```bash
CREATE DATABASE next;
CREATE USER next_user WITH ENCRYPTED PASSWORD 'Abcd@1234';
GRANT ALL PRIVILEGES ON DATABASE next TO next_user;
\q
```

*   Nếu project dùng **Prisma** hoặc **Drizzle**, chạy migrate:<br>

    ```bash
    pnpm prisma migrate deploy  # hoặc pnpm drizzle-kit push:pg
    ```


* Nếu dùng raw pg client, đảm bảo kết nối trong code.

## **Chạy Next.js server với pm2**

Cài pm2 global nếu chưa:

```bash
pnpm add -g pm2
```

Tạo file `ecosystem.config.js` ở root project (tốt hơn cho zero-downtime):

```js
module.exports = {
  apps: [
    {
      name: 'chomusuke-demo-next',
      script: 'pnpm',
      args: 'start',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
      },
      env_production: {
        PORT: 3001,
      },
    },
  ],
};
```

Chạy:

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup  # Auto start sau reboot
pm2 monit    # Theo dõi
```

## **Cấu hình Nginx reverse proxy**

```bash
sudo nano /etc/nginx/sites-available/chomusuke-demo-next.conf
```

Paste nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name next.chomusuke.site;

    location ^~ /_next/static/ {
        alias /var/www/chomusuke-demo-next/.next/static/;
        expires 1y;
        access_log off;
        add_header Cache-Control "public, immutable";
    }
    
    location = /favicon.ico {
        alias /var/www/chomusuke-demo-next/public/favicon.ico;
        expires 1y;
        access_log off;
        add_header Cache-Control "public, immutable";
    }

    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        access_log off;
        add_header Cache-Control "public";
    }

    access_log /var/log/nginx/chomusuke-demo-next.access.log;
    error_log /var/log/nginx/chomusuke-demo-next.error.log;
}
```

> Chú ý move favicon.ico vào /public để tránh lỗi nginx không serve được favicon.

Lưu file → enable site

{% code overflow="wrap" %}
```bash
sudo ln -s /etc/nginx/sites-available/chomusuke-demo-next.conf /etc/nginx/sites-enabled/
```
{% endcode %}

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

## **Permissions và ownership**

{% code overflow="wrap" %}
```bash
sudo chown -R www-data:www-data /var/www/chomusuke-demo-next/.next
sudo chmod -R 775 /var/www/chomusuke-demo-next/.next
```
{% endcode %}

## **Kiểm tra và debug lỗi phổ biến**

Truy cập [https://next.chomusuke.site](https://next.chomusuke.site/?referrer=grok.com) (sau khi trỏ DNS).

* **502**: pm2 down → `pm2 status`, port sai → `netstat -tuln | grep 3001`
* **504**: API route chậm → `pm2 logs chomusuke-demo-next`
* **ISR không regenerate**: Redis connection → `redis-cli KEYS 'nextjs:*'`
* **Hydration error**: NEXT\_PUBLIC\_ vars mismatch
* **DB connection fail**: Test `psql $DATABASE_URL`
* Logs: `pm2 logs chomusuke-demo-next`, `tail -f /var/log/nginx/chomusuke-demo-next.error.log`

## **Update code sau này**

```bash
cd /var/www/chomusuke-demo-next
git pull origin main
pnpm install
pnpm build
pm2 reload chomusuke-demo-next  # Zero-downtime nếu cluster
sudo systemctl reload nginx
```

Vậy là xong! Ứng dụng Next.js hybrid của bạn đã live với folder/config theo tên repo, PostgreSQL database riêng next + user next\_user, Redis chung instance (độc lập data qua prefix), pm2 quản lý process, và Nginx reverse proxy.
