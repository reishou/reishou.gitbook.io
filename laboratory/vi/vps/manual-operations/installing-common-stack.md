---
icon: globe
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/installing-common-stack
---

# Cài Đặt Stack Phổ Biến

Trước khi đi vào deploy từng loại ứng dụng cụ thể (static site Astro, Laravel, NextJS, Go app), chúng ta cần chuẩn bị một server Ubuntu với **các thành phần nền tảng chung** mà hầu hết các dự án sẽ dùng. Phần này giúp bạn chỉ cài một lần, sau đó tái sử dụng cho các ví dụ sau.

Chúng ta sẽ cài:&#x20;

* Các gói cơ bản + utils
* Nginx (web server/reverse proxy)
* PHP 8.3/8.4 + PHP-FPM + extensions phổ biến (cho Laravel)
* Composer (dependency manager cho PHP)
* Node.js LTS (cho NextJS và Astro build)
* Go (cho ứng dụng Golang)
* Supervisor (quản lý background process như queue Laravel)

**Lưu ý**: Tất cả lệnh dưới đây chạy với quyền **root** hoặc dùng `sudo`.

## Cập nhật hệ thống & cài các gói cơ bản

Bước đầu tiên luôn luôn là update hệ thống và cài các công cụ cần thiết.

{% code overflow="wrap" %}
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip build-essential ca-certificates lsb-release gnupg2
```
{% endcode %}

## Cài đặt Nginx (từ official repo – khuyến nghị stable)

Dùng repo chính thức từ nginx.org để có phiên bản mới và ổn định hơn repo mặc định của Ubuntu.

Cài prerequisite

```bash
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring -y
```

Import signing key

{% code overflow="wrap" %}
```bash
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```
{% endcode %}

Thêm repo stable (khuyến nghị cho production)

{% code overflow="wrap" %}
```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
```
{% endcode %}

Pin priority để ưu tiên repo nginx.org

{% code overflow="wrap" %}
```bash
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900" | sudo tee /etc/apt/preferences.d/99nginx
```
{% endcode %}

Update & install

```bash
sudo apt update
sudo apt install nginx -y
```

Kiểm tra

```bash
nginx -v
systemctl status nginx
```

Sau khi cài xong, bạn có thể truy cập IP server qua browser → thấy trang "Welcome to nginx!" là ok.

## Cài PHP + PHP-FPM + extensions phổ biến (dùng PPA ondrej/php)

Ubuntu 24.04 mặc định có PHP 8.3, nhưng để có PHP 8.4 (hoặc mới hơn) và đầy đủ extensions, dùng PPA của Ondřej Surý (rất uy tín trong cộng đồng PHP).

Thêm PPA

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

Cài PHP 8.4 + FPM + các extension phổ biến cho Laravel (có thể thay 8.4 bằng 8.3 nếu muốn)

```bash
sudo apt install -y php8.4 php8.4-fpm php8.4-cli php8.4-mbstring php8.4-xml php8.4-curl \
    php8.4-zip php8.4-gd php8.4-intl php8.4-bcmath
```

Kiểm tra php

```bash
php -v
php-fpm8.4 -v
```

Cài Composer (global)

```bash
curl -sS https://getcomposer.org/installer -o composer-setup.php
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm composer-setup.php
```

Kiểm tra composer

```bash
composer --version
```

## Cài Node.js LTS (qua Nodesource – apt style)

Thêm repo cho Node.js LTS (hiện tại thường là 20.x hoặc 22.x)

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
```

Cài đặt

```bash
sudo apt update
sudo apt install -y nodejs
sudo npm install -g pnpm
```

Kiểm tra

```bash
node -v
npm -v
pnpm -v
```

## Cài Go (Golang) – dùng tarball official (khuyến nghị)

Repo Ubuntu thường cũ, nên official khuyên dùng tarball để luôn có phiên bản mới nhất.

Tải phiên bản mới nhất (thay VERSION bằng phiên bản hiện tại, ví dụ 1.24.x)

```bash
VERSION=1.25.6  # <-- check tại https://go.dev/dl/ để lấy version mới nhất
wget https://go.dev/dl/go${VERSION}.linux-amd64.tar.gz
```

Extract vào `/usr/local`

```bash
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go${VERSION}.linux-amd64.tar.gz
```

Thêm vào `PATH` (vĩnh viễn)

```bash
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee -a /etc/profile
source /etc/profile
```

Kiểm tra

```bash
go version
```

## Cài Supervisor – quản lý process background

Rất hữu ích cho Laravel queue worker

```bash
sudo apt install supervisor -y
```

Chỉnh file config (thường là `/etc/supervisor/supervisord.conf` hoặc `/etc/supervisord.conf`)

```bash
sudo nano /etc/supervisor/supervisord.conf
```

Tìm phần \[unix\_http\_server] (nếu không có thì thêm vào):

{% code overflow="wrap" %}
```bash
[unix_http_server]
file=/var/run/supervisor.sock          ; đường dẫn socket (giữ nguyên hoặc thay nếu khác)
chmod=0770                             ; cho phép owner + group đọc/ghi
chown=root:www-data                    ; thay www-data bằng group bạn muốn
```
{% endcode %}

Save rồi restart

```bash
sudo systemctl restart supervisor
```

Kiểm tra

```bash
supervisorctl status
```

Hoặc nếu bạn thích pm2 cho Node.js (sẽ dùng ở phần NextJS)

```bash
sudo npm install -g pm2
```

## Kết thúc – Kiểm tra toàn bộ stack

```bash
nginx -v
php -v
composer --version
node -v && npm -v
go version
supervisorctl status
```

Nếu tất cả đều hiện version → bạn đã có một server common stack sẵn sàng cho các phần deploy tiếp theo.
