---
icon: golang
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/deploying-golang-application-manually
---

# Deploy Thủ Công Ứng Dụng Golang

## Build binary trên máy local

Trên máy dev (Mac/Windows/Linux – nơi đã clone repo và chạy `go mod tidy`):

```bash
# Vào folder project
cd chomusuke-demo-go

# Build cho Linux amd64 (Ubuntu 24.04 phổ biến dùng amd64)
GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o chomusuke-go ./cmd/web
```

* Binary `chomusuke-go` sẽ nằm ngay trong folder hiện tại.
* Nếu VPS là ARM64 (hiếm với Ubuntu 24 desktop/server thông thường), đổi thành `GOARCH=arm64`.
*   Test nhanh (optional):<br>

    ```bash
    source .env && ./chomusuke-go
    ```

    \
    → Truy cập `localhost:8000` để kiểm tra.

## Copy binary lên VPS

Từ máy local:

{% code overflow="wrap" %}
```bash
# Tạo folder trên VPS nếu chưa có (chạy lệnh này trước nếu cần)
# ssh vps-user@your-vps-ip "sudo mkdir -p /var/www/chomusuke-demo-go && sudo chown vps-user:vps-user /var/www/chomusuke-demo-go"

scp chomusuke-go vps-user@your-vps-ip:/var/www/chomusuke-demo-go/
```
{% endcode %}

(Thay `your-vps-ip` bằng IP thực tế. Folder `/var/www/chomusuke-demo-go` sẽ dùng để đồng bộ với các project web khác của bạn.)

## Setup trên VPS

SSH vào VPS với user `vps-user`:

```bash
cd /var/www/chomusuke-demo-go
chmod +x chomusuke-go
```

Tạo file env (nếu chưa có):

```bash
nano .env
```

Nội dung ví dụ:

```ini
APP_ENVIRONMENT=production
APP_ENCRYPTIONKEY=your_very_long_random_key_here   # generate: openssl rand -base64 32
DATABASE_CONNECTION=postgres://postgres:yourpass@127.0.0.1:5432/chomusuke
HTTP_PORT=8000
```

Test chạy tay:

```bash
source .env && ./chomusuke-go
```

→ `curl localhost:8000` hoặc browser `VPS-IP:8000` để kiểm tra. Dừng bằng `Ctrl+C`.

## Chạy như systemd service

Tạo file service:

```bash
sudo nano /etc/systemd/system/chomusuke.service
```

Nội dung:

```ini
[Unit]
Description=Chomusuke Go App - go.chomusuke.site
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/www/chomusuke-demo-go
EnvironmentFile=/var/www/chomusuke-demo-go/.env
ExecStart=/var/www/chomusuke-demo-go/chomusuke-go
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Apply:

```bash
sudo systemctl daemon-reload
sudo systemctl start chomusuke
sudo systemctl enable chomusuke
```

Kiểm tra:

```bash
sudo systemctl status chomusuke
journalctl -u chomusuke -f   # log real-time
```

#### 5. Proxy qua Nginx (dùng domain go.chomusuke.site)

Tạo config:

```bash
sudo nano /etc/nginx/sites-available/chomusuke-demo-go.conf
```

Nội dung:

```nginx
server {
    listen 80;
    server_name go.chomusuke.site;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Lưu file → enable site

```bash
sudo ln -s /etc/nginx/sites-available/chomusuke-demo-go.conf /etc/nginx/sites-enabled/
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

App giờ đã truy cập được qua [http://go.chomusuke.site](http://go.chomusuke.site) (HTTPS có thể thêm sau khi cần).

## Update nhanh khi có code mới

Trên local: build binary mới → scp lại:

```bash
scp chomusuke-go vps-user@your-vps-ip:/var/www/chomusuke-demo-go/
```

Trên VPS:

```bash
cd /var/www/chomusuke-demo-go
sudo systemctl restart chomusuke
```

Xong, downtime chỉ vài giây.

#### Lưu ý

* Backup `.env` và database trước khi update.
* Nếu gặp lỗi permission folder: `sudo chown -R vps-user:vps-user /var/www/chomusuke-demo-go`
* Theo dõi log: `journalctl -u chomusuke -f`
