---
icon: shield-check
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/setting-up-https-with-ssl-cloudflare-letsencrypt
---

# Cấu Hình HTTPS Với SSL (Cloudflare hoặc Let's Encrypt)

Tiếp nối bài trỏ domain về VPS qua Cloudflare, giờ ta bật **HTTPS** cho subdomain `go.chomusuke.site` (và domain chính nếu cần). Nginx hiện tại chỉ listen port 80 (HTTP), ta sẽ thêm port 443 (HTTPS) và cert SSL.

Có 2 cách chính:

* **Cloudflare Origin CA** (dễ nhất, miễn phí, nhanh, phù hợp khi dùng proxy Cloudflare orange cloud).
* **Let's Encrypt** (cert công khai, dùng khi tắt proxy Cloudflare hoặc cần Full strict mode).

## **Chuẩn bị chung**

* Nginx đã cài và config proxy đến app Go (port 8080) như bài trước.
*   Folder lưu cert: /etc/ssl/cloudflare/ (tạo nếu chưa có).<br>

    ```bash
    sudo mkdir -p /etc/ssl/cloudflare
    sudo chmod 700 /etc/ssl/cloudflare   # Chỉ root đọc được
    ```


*   Backup config Nginx cũ:<br>

    ```bash
    sudo cp /etc/nginx/sites-available/chomusuke /etc/nginx/sites-available/chomusuke.bak
    ```

## Cloudflare Origin CA (Khuyến nghị khi dùng proxy Cloudflare)

Cloudflare cấp cert riêng cho origin server (VPS), dùng để kết nối giữa Cloudflare → VPS (Full strict mode).

{% stepper %}
{% step %}
### **Tạo Origin Certificate trên Cloudflare**

* Đăng nhập dash.cloudflare.com → Chọn domain chomusuke.site.
* Tab **SSL/TLS** → **Origin Server** → **Create Certificate**.
* Chọn:
  * Certificate validity: 15 năm (mặc định).
  * Hostnames: `*.chomusuke.site` (wildcard để dùng cho tất cả subdomain) hoặc cụ thể `go.chomusuke.site`.
  * Key type: RSA hoặc ECC (RSA ổn định hơn).
* Nhấn **Create** → Cloudflare sinh ra:
  * Origin Certificate (file `.crt`).
  * Private Key (file `.key`).
* Copy 2 nội dung này (không lưu file tạm trên máy, copy trực tiếp).
{% endstep %}

{% step %}
### **Lưu cert vào VPS Trên VPS (SSH với vps-user)**

{% code overflow="wrap" %}
```bash
sudo nano /etc/ssl/cloudflare/chomusuke.site.crt
# Paste toàn bộ Origin Certificate (bắt đầu -----BEGIN CERTIFICATE----- đến -----END CERTIFICATE-----)
# Ctrl+O → Enter → Ctrl+X để save

sudo nano /etc/ssl/cloudflare/chomusuke.site.key
# Paste Private Key (-----BEGIN PRIVATE KEY----- đến -----END PRIVATE KEY-----)
# Save tương tự
```
{% endcode %}

Set quyền:

```bash
sudo chmod 600 /etc/ssl/cloudflare/chomusuke.site.*
```
{% endstep %}

{% step %}
### **Cập nhật config Nginx**

Chỉnh file `/etc/nginx/sites-available/chomusuke`:

```nginx
server {
    listen 80;
    server_name go.chomusuke.site;

    # Redirect HTTP → HTTPS (tùy chọn, nhưng khuyến nghị)
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name go.chomusuke.site;

    ssl_certificate /etc/ssl/cloudflare/chomusuke.site.crt;
    ssl_certificate_key /etc/ssl/cloudflare/chomusuke.site.key;

    # Proxy đến app Go
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Kiểm tra & restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```
{% endstep %}

{% step %}
### **Bật Full (strict) trên Cloudflare**

* Cloudflare → SSL/TLS → Overview → Set **Encryption mode** thành **Full (strict)**.
* Đợi 5–10 phút → Truy cập [https://go.chomusuke.site](https://go.chomusuke.site) → OK (padlock xanh).
{% endstep %}
{% endstepper %}

## Let's Encrypt (Certbot) – Nếu không dùng proxy Cloudflare hoặc muốn cert công khai

Nếu bạn tắt proxy (grey cloud) cho record A của go.chomusuke.site, dùng Certbot để lấy cert miễn phí từ Let's Encrypt.

{% stepper %}
{% step %}
### **Bước 1: Cài Certbot**

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```
{% endstep %}

{% step %}
### **Chạy Certbot tự động config Nginx**

```bash
sudo certbot --nginx -d go.chomusuke.site
```

* Nhập email (cho thông báo renew).
* Đồng ý terms.
* Chọn redirect HTTP → HTTPS (nên chọn 2: Redirect).
* Certbot tự thêm listen 443, ssl\_certificate, và redirect 80 → 443.
{% endstep %}

{% step %}
### Kiểm tra

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Cert lưu tự động ở `/etc/letsencrypt/live/go.chomusuke.site/fullchain.pem` và `privkey.pem`. Certbot tự renew (cron job hàng ngày).
{% endstep %}

{% step %}
### **Nếu dùng Cloudflare**

* Set Encryption mode: **Full** (không strict, vì cert Let's Encrypt là public, không phải Origin CA).
* Hoặc giữ Full strict nếu bạn muốn, nhưng Origin CA vẫn tốt hơn.
{% endstep %}
{% endstepper %}

## Kiểm tra HTTPS

* Truy cập [https://go.chomusuke.site](https://go.chomusuke.site) → Padlock xanh, no warning.
* Kiểm tra cert: Click padlock → Certificate → Issuer là Cloudflare hoặc Let's Encrypt.
* Test SSL: [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/) → Nhập go.chomusuke.site → Chờ kết quả (nên A hoặc A+).

#### **Lưu ý**

* Nếu dùng Cloudflare proxy: Luôn ưu tiên Origin CA + Full strict → an toàn hơn (IP VPS ẩn hoàn toàn).
* Renew Let's Encrypt: Tự động, nhưng check `sudo certbot renew --dry-run` nếu lo.
* Backup cert: `sudo cp -r /etc/ssl/cloudflare /backup/` hoặc `/etc/letsencrypt`.
* Lỗi phổ biến: "nginx: \[emerg] SSL: error:0B080074:x509 certificate routines" → Kiểm tra key/crt khớp, không paste thừa dòng.
