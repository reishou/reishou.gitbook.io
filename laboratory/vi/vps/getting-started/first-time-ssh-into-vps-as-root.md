---
icon: key
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/getting-started/first-time-ssh-into-vps-as-root
---

# Lần Đầu SSH Vào VPS Với Tài Khoản Root

Khi mới nhận VPS từ nhà cung cấp (DigitalOcean, Vultr, Linode, v.v.), bước đầu tiên là kết nối vào server qua SSH bằng tài khoản **root**. Đây là cách duy nhất để quản lý server từ xa một cách an toàn và hiệu quả.

Bài viết này hướng dẫn chi tiết cách SSH lần đầu trên **macOS** và **Linux** (Ubuntu/Debian desktop), đồng thời tập trung vào hệ điều hành server **Ubuntu/Debian**.

## Thông tin cần chuẩn bị

Từ nhà cung cấp VPS, bạn sẽ nhận được:

* **Địa chỉ IP** của VPS (ví dụ: `123.45.67.89`)
* **Tài khoản đăng nhập**: `root`
* **Mật khẩu root** (gửi qua email hoặc hiển thị trên dashboard)
* **Port SSH**: Mặc định là `22` (nếu nhà cung cấp thay đổi thì sẽ thông báo rõ ràng)

{% hint style="info" %}
**Lưu ý bảo mật rất quan trọng**: Đăng nhập root trực tiếp bằng mật khẩu chỉ nên thực hiện **lần đầu tiên**. Ngay sau khi kết nối thành công, bạn cần thực hiện các bước bảo mật cơ bản (tạo user thường có quyền sudo, thiết lập SSH key, tắt root login, v.v.). Những bước này sẽ được hướng dẫn chi tiết ở các bài tiếp theo.
{% endhint %}

## Kết nối SSH trên macOS và Linux

Cả macOS và Linux đều có Terminal tích hợp sẵn, việc kết nối rất đơn giản.

{% stepper %}
{% step %}
Mở **Terminal**:

* macOS: Nhấn `Cmd + Space` → gõ "Terminal"
* Linux (Ubuntu/Debian): Nhấn `Ctrl + Alt + T`
{% endstep %}

{% step %}
Gõ lệnh kết nối:

```bash
ssh root@123.45.67.89
```

(Thay `123.45.67.89` bằng IP thực tế của VPS)
{% endstep %}

{% step %}
Lần đầu kết nối, hệ thống sẽ hỏi xác nhận fingerprint:

```bash
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

→ Gõ `yes` rồi nhấn Enter.
{% endstep %}

{% step %}
Nhập mật khẩu root:

* Khi gõ mật khẩu sẽ **không hiển thị** bất kỳ ký tự nào (kể cả dấu chấm hoặc sao). Đây là tính năng bảo mật bình thường.
* Gõ xong thì nhấn Enter.
{% endstep %}
{% endstepper %}

Nếu thành công, bạn sẽ thấy prompt giống như:

```bash
root@hostname:~#
```

**Trường hợp port SSH không phải 22** (một số nhà cung cấp đổi port để tăng bảo mật):

```bash
ssh root@123.45.67.89 -p 2222
```

(Thay `2222` bằng port thực tế)

## Các lỗi thường gặp và cách khắc phục

* **Connection timed out** hoặc **No route to host**: Kiểm tra lại IP/port, đảm bảo VPS đang chạy và mạng internet của bạn ổn định. Có thể firewall nhà cung cấp chưa mở port SSH.
* **Permission denied (publickey, password)**: Sai mật khẩu hoặc nhà cung cấp đã tắt login bằng password. Thử copy-paste mật khẩu cẩn thận hoặc liên hệ support để kiểm tra.
* **Connection refused**: Port SSH sai hoặc service SSH chưa chạy (hiếm gặp với VPS mới).

Nếu thử nhiều lần sai mật khẩu và bị khóa tạm thời, hãy liên hệ hỗ trợ nhà cung cấp để reset hoặc sử dụng console khẩn cấp (VNC/KVM) nếu có.
