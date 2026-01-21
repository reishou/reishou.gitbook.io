---
icon: cloudflare
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/manual-operations/pointing-your-domain-to-vps-with-cloudflare
---

# Trỏ Domain Về VPS Qua Cloudflare

**Cloudflare** là dịch vụ DNS trung gian miễn phí, giúp trỏ domain/subdomain về VPS nhanh, an toàn, có CDN, bảo vệ DDoS cơ bản, và tự động HTTPS (Universal SSL). Trong bài này, ta trỏ subdomain `go.chomusuke.site` về VPS (IP của bạn), với proxy bật (orange cloud) để tận dụng lợi ích Cloudflare.

{% stepper %}
{% step %}
### **Chuẩn bị**

* Tài khoản Cloudflare (đăng ký miễn phí tại dash.cloudflare.com).
* Domain chính (`chomusuke.site`) đã đăng ký ở đâu đó (Namecheap, GoDaddy, Mat Bao, v.v.).
* IP VPS (kiểm tra bằng `curl ifconfig.me` hoặc trong panel VPS).
* Nginx đã cài và config proxy đến app Go (port 8080) như bài trước.
{% endstep %}

{% step %}
### **Thêm domain vào Cloudflare**

* Đăng nhập `dash.cloudflare.com`.
* Nhấn **Add a site** (hoặc "Add site").
* Nhập domain chính: **chomusuke.site** (không nhập subdomain).
* Chọn plan **Free** → Continue.
* Cloudflare sẽ quét DNS records hiện tại (nếu có) → Skip hoặc Continue.
* Cloudflare đưa ra 2 nameservers (ví dụ: `ns1.cloudflare.com` và `ns2.cloudflare.com`) → Copy chính xác.
{% endstep %}

{% step %}
### **Trỏ nameservers về Cloudflare tại nhà cung cấp domain**

* Đăng nhập panel quản lý domain (nơi bạn mua `chomusuke.site`).
* Tìm phần **Nameservers** hoặc **DNS Nameservers**.
* Xóa nameservers cũ (thường là default của registrar).
* Thêm 2 nameservers của Cloudflare (paste chính xác, không thêm www hoặc http).
* Save → Đợi propagation (thường 5–60 phút, tối đa 24h).
* Kiểm tra: Vào Cloudflare dashboard → Overview → Nếu thấy status **Active**, OK.
{% endstep %}

{% step %}
### **Thêm DNS record cho subdomain go.chomusuke.site**

* Trong Cloudflare dashboard → Chọn domain chomusuke.site → Tab **DNS**.
* Nhấn **Add record**.
* Cấu hình:
  * Type: **A**
  * Name: **go** (Cloudflare sẽ tự thêm .chomusuke.site)
  * IPv4 address: **IP của VPS** (ví dụ 123.456.78.90)
  * Proxy status: **Bật (orange cloud)** – Để Cloudflare proxy traffic (ẩn IP thật, bật CDN, HTTPS tự động).
  * TTL: **Auto**
* Save.
* (Optional) Nếu muốn [www.go.chomusuke.site](http://www.go.chomusuke.site): Thêm record CNAME
  * Type: **CNAME**
  * Name: [**www.go**](http://www.go)
  * Target: **go.chomusuke.site**
  * Proxy: Bật orange cloud.
{% endstep %}

{% step %}
### **Bật HTTPS tự động (Universal SSL)**

* Trong Cloudflare → Tab **SSL/TLS** → Overview.
* Set **Encryption mode** thành **Full (strict)** (an toàn nhất, yêu cầu VPS có cert tự ký hoặc Let's Encrypt).
* Cloudflare sẽ tự cấp cert miễn phí cho domain (khi proxy bật).
{% endstep %}

{% step %}
### **Kiểm tra & hoàn tất**

* Đợi DNS propagation (dùng [https://www.whatsmydns.net/](https://www.whatsmydns.net/) kiểm tra A record cho go.chomusuke.site).
* Truy cập [http://go.chomusuke.site](http://go.chomusuke.site) hoặc [https://go.chomusuke.site](https://go.chomusuke.site) → Nếu thấy app Go chạy → thành công.
* Kiểm tra IP ẩn: Dùng [https://whatismyipaddress.com](https://whatismyipaddress.com) → Nên thấy IP Cloudflare (không phải IP VPS thật).
* Log Cloudflare: Tab **Analytics** → Xem traffic, attack, cache hit.
{% endstep %}
{% endstepper %}

#### **Lưu ý**

* Proxy bật (orange cloud): Ẩn IP VPS, chống DDoS, cache static, HTTPS free. Nhưng nếu app dùng WebSocket hoặc non-HTTP → có thể cần tắt proxy (grey cloud) cho record đó.
* Nếu domain chính đã trỏ nameservers về Cloudflare → tất cả subdomain đều quản lý ở đây (không cần trỏ riêng).
* Propagation chậm: Xóa cache browser/DNS (Ctrl+Shift+R) hoặc dùng incognito.
* Bảo mật thêm: Trong Cloudflare → Firewall → Bật WAF rules, rate limiting nếu cần.
* Nếu subdomain không resolve: Kiểm tra nameservers đã active chưa (Cloudflare Overview).

Xong! Subdomain `go.chomusuke.site` giờ đã trỏ về VPS qua Cloudflare, an toàn và nhanh hơn.
