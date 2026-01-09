---
icon: key
metaLinks:
  alternates:
    - /broken/spaces/fZMM9Pd2vdURETYcbaWM/pages/AolChTehjqM2xgWXXgl4
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

## Kiểm tra nhanh server

#### Kiểm tra thời gian hoạt động của server (uptime)

```bash
uptime
```

**Output ví dụ**:

```bash
14:35:22 up 5 days,  3:12,  1 user,  load average: 0.15, 0.10, 0.08
```

**Giải thích các ký hiệu**:

* `14:35:22`: Thời gian hiện tại trên server.
* `up 5 days, 3:12`: Server đã chạy liên tục **5 ngày 3 giờ 12 phút** kể từ lần boot cuối.
* `1 user`: Số người dùng đang đăng nhập (thường là bạn qua SSH).
* `load average: 0.15, 0.10, 0.08`: Load trung bình trong 1, 5 và 15 phút gần nhất. Giá trị càng thấp càng tốt. Với server 1 CPU core, load < 1.0 là lý tưởng; > số core thì bắt đầu có dấu hiệu quá tải.

**Lệnh thay thế (đọc dễ hơn)**:

```bash
uptime -p
```

→ Output: `up 5 days, 3 hours, 12 minutes`

#### Kiểm tra dung lượng đĩa (Disk usage)

```bash
df -h
```

**Output ví dụ**:

```bash
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        25G  4.2G   19G  19% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
/dev/sda15      105M  5.3M  100M   6% /boot/efi
```

**Giải thích các cột**:

* `Size`: Dung lượng tổng của phân vùng.
* `Used`: Đã sử dụng.
* `Avail`: Còn trống.
* `Use%`: Phần trăm đã dùng (nên giữ dưới 85-90% để tránh đầy đĩa).
* `Mounted on`: Điểm gắn (thường / là root filesystem – quan trọng nhất).

#### Kiểm tra bộ nhớ RAM và Swap

```bash
free -h
```

**Output ví dụ**:

```bash
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       580Mi       2.9Gi       100Mi       350Mi       3.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

**Giải thích các cột**:

* `total`: Tổng RAM/Swap.
* `used`: Đang sử dụng thực tế.
* `free`: Còn trống hoàn toàn.
* `buff/cache`: Bộ nhớ dùng cho cache/buffer (có thể giải phóng khi cần).
* `available`: Lượng RAM thực tế còn dùng được cho ứng dụng mới (quan trọng nhất).
* `Swap`: Nếu `used` > 0 nhiều → có thể server đang thiếu RAM.

#### Xem nhanh tình trạng CPU, RAM, process (top hoặc htop)

```bash
top
```

hoặc (khuyến nghị, đẹp và dễ dùng hơn):

```bash
htop
```

(Nếu chưa có htop: `apt update && apt install htop -y`)

**Trong htop/top**:

* Dòng đầu tiên: uptime, load average, số task/process.
* CPU bar: `%us` (user), `%sy` (system), `%id` (idle – idle càng cao càng tốt).
* Mem: Tương tự free -h.
* Swap: Nên gần 0.
* Danh sách process: Sắp xếp theo CPU/MEM bằng phím P/M.

#### Tóm tắt lệnh kiểm tra nhanh (copy-paste một lần)

```bash
uptime && echo "" && df -h && echo "" && free -h && echo "" && top -bn1 | head -n 15
```

Bạn có thể chạy các lệnh này ngay sau khi login để xác nhận server ổn định trước khi tiến hành cài đặt tiếp.
