---
icon: screwdriver-wrench
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/getting-started/basic-server-setup
---

# Cài Đặt Cơ Bản Cho Server

Sau khi cài xong Ubuntu Server, hầu hết mọi người đều quên hoặc làm qua loa phần cấu hình cơ bản. Đây chính là giai đoạn quyết định server của bạn sẽ **ổn định, an toàn và dễ quản lý** trong thời gian dài hay không.

Dưới đây là các bước mình luôn làm trên **mọi server mới** (VPS, dedicated, home-lab…).

## Cập nhật hệ thống đầy đủ (bắt buộc)

```bash
# Đưa hệ thống về trạng thái mới nhất
sudo apt update
sudo apt upgrade -y

# Cài thêm các bản vá kernel/security nếu có (rất khuyến khích)
sudo apt dist-upgrade -y

# Dọn dẹp gói thừa, cache...
sudo apt autoremove -y
sudo apt autoclean

# Khởi động lại nếu kernel đã được cập nhật (thường nên restart)
sudo reboot
```

**Lưu ý**: Trên Ubuntu 24.04+, `unattended-upgrades` thường đã bật sẵn → tự động cài các bản vá bảo mật. Bạn có thể kiểm tra & tùy chỉnh file `/etc/apt/apt.conf.d/50unattended-upgrades`.

## Đặt đúng Timezone & đồng bộ thời gian

Hầu hết các log, cron job, certificate sẽ dựa vào thời gian hệ thống. Đặt sai timezone rất dễ gây nhầm lẫn khi debug.

```bash
# Xem danh sách timezone có sẵn
timedatectl list-timezones | grep Asia   # ví dụ tìm múi giờ Việt Nam

# Đặt timezone (múi giờ Việt Nam phổ biến nhất)
sudo timedatectl set-timezone Asia/Ho_Chi_Minh

# Kiểm tra lại
timedatectl
```

Kết quả mong muốn:

```bash
Local time: Mon 2026-01-12 17:45:22 +07
  Universal time: Mon 2026-01-12 10:45:22 UTC
        RTC time: Mon 2026-01-12 10:45:22
       Time zone: Asia/Ho_Chi_Minh (+07, +0700)
```

**Bonus**: Hệ thống Ubuntu/Debian hiện đại đã dùng **systemd-timesyncd** (NTP client nhẹ) → tự động đồng bộ thời gian rồi, không cần cài `ntp` nữa trừ trường hợp đặc biệt.

## Tạo Swap File (rất nên có, đặc biệt VPS RAM thấp)

Hầu hết các VPS hiện nay đều dùng SSD/NVMe, nên **swap file** là lựa chọn tốt hơn swap partition (dễ thay đổi kích thước).

**Khuyến nghị kích thước swap** (thực tế sử dụng phổ biến):

| RAM vật lý | Swap đề xuất   | Ghi chú                                         |
| ---------- | -------------- | ----------------------------------------------- |
| ≤ 2GB      | 2–4GB          | Bắt buộc, tránh OOM killer                      |
| 2–8GB      | 2–4GB          | An toàn cho hầu hết web app nhỏ–trung bình      |
| 8–16GB     | 2–4GB hoặc 8GB | Tùy workload (database, java thì nên nhiều hơn) |
| > 16GB     | 2–8GB          | Chủ yếu làm safety net, ít dùng đến             |

Ví dụ tạo **4GB swap** (phổ biến & cân bằng nhất hiện nay):

```bash
# Tạo file swap 4GB
sudo fallocate -l 4G /swapfile
# Hoặc dùng dd nếu fallocate không được (ít gặp)
# sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress

sudo chmod 600 /swapfile           # Bảo mật
sudo mkswap /swapfile              # Format thành swap
sudo swapon /swapfile              # Kích hoạt ngay

# Kiểm tra
free -h

# Làm vĩnh viễn (quan trọng!)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Tối ưu swappiness (rất khuyến khích trên server)
# Giá trị 10–30 là tốt nhất cho hầu hết trường hợp
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Tùy chọn nâng cao (giảm ghi đè lên SSD)
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Các bước cơ bản khác nên làm luôn (tùy chọn nhưng rất đáng)

```bash
# Đặt hostname dễ nhận biết (ví dụ: web1, api-stg, db-prod...)
sudo hostnamectl set-hostname web-server-01

# Cài một số công cụ tiện ích (rất nên)
sudo apt install -y curl wget git htop ncdu tree unzip zip net-tools \
    ca-certificates apt-transport-https software-properties-common
```

#### Tóm tắt thứ tự khuyến nghị

1. Update & upgrade toàn bộ hệ thống → reboot nếu cần
2. Đặt đúng timezone (Asia/Ho\_Chi\_Minh)
3. Tạo & cấu hình swap file + swappiness
4. Đặt hostname rõ ràng
5. Cài các gói tiện ích cơ bản

Sau khi hoàn thành các bước trên, server của bạn đã ở trạng thái **sạch, ổn định và an toàn tương đối** để bắt đầu cài đặt ứng dụng.
