---
icon: a
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/infrastructure-as-code/getting-started-with-ansible-for-vps
---

# Giới thiệu & Cài đặt Ansible Cho VPS

## Giới thiệu

Nếu bạn đang quản lý một hoặc vài VPS (DigitalOcean, Vultr, Lightsail, hay server tự host), chắc hẳn bạn đã quen với việc SSH vào rồi gõ lệnh thủ công: update package, cài Nginx, config firewall, deploy app... Làm đi làm lại nhiều lần thì mệt, dễ sai sót, và khó quản lý nếu có thêm server.

**Ansible** giúp giải quyết triệt để:

* Bạn viết "code" đơn giản bằng YAML để mô tả **server phải như thế nào**.
* Ansible tự động SSH vào VPS và thực hiện, đảm bảo đúng trạng thái mong muốn.
* Chạy lại nhiều lần cũng không sao (idempotent) – an toàn, không làm hỏng gì thêm.
* Không cần cài phần mềm gì lên VPS (agentless) → rất nhẹ.

Series này dành cho người **mới bắt đầu từ con số 0** (như mình lúc đầu), tập trung thực hành trên VPS cá nhân/homelab. Chúng ta sẽ đi từng bước:

* Cài đặt & lệnh đầu tiên
* Viết playbook cơ bản
* Tạo role modular (Nginx + PHP, Node.js...)
* Deploy app thật: Astro (static), Laravel, Next.js
* Tự động hóa bằng GitHub Actions

Repo minh họa toàn bộ series:\
[https://github.com/reishou/chomusuke-vps-ansible](https://github.com/reishou/chomusuke-vps-ansible)

## Tại sao chọn Ansible thay vì Bash script?

* **Declarative**: Mô tả kết quả cuối cùng, không cần viết từng bước lệnh.
* **Idempotent**: Chạy lại không thay đổi nếu đã đạt trạng thái mong muốn.
* Dễ đọc, dễ debug (YAML thay vì shell script dài).
* Hàng trăm module sẵn có (apt, file, service, git, template...).
* Dễ tích hợp Git → version control cho automation.

## Yêu cầu trước khi bắt đầu

* Máy local (control node): macOS, Linux, hoặc Windows (dùng WSL).
* VPS chạy Ubuntu/Debian (khuyến nghị 22.04 hoặc 24.04 LTS).
* SSH key đã copy vào VPS (không dùng password để an toàn).
* Python 3 cài sẵn trên cả local và VPS.

## Cài đặt Ansible trên máy local

Ansible chỉ cài trên máy bạn, không cần cài trên VPS.

**Trên macOS (Homebrew – khuyến nghị)**

{% code overflow="wrap" %}
```bash
# Cài Homebrew nếu chưa có
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Cài Ansible
brew install ansible
```
{% endcode %}

Hoặc dùng pip:

```bash
pip3 install --user ansible
```

**Trên Linux (Ubuntu/Debian)**

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

Hoặc pip:

```bash
sudo apt install python3-pip
pip3 install ansible
```

**Trên Windows (dùng WSL)**

1. Mở PowerShell (admin): `wsl --install` → cài Ubuntu.
2. Vào terminal Ubuntu trong WSL.
3. Cài như trên Linux.

Kiểm tra sau khi cài:

```bash
ansible --version
```

Kết quả ví dụ:

```bash
ansible [core 2.16.x] 
  config file = None
  ...
```

Nếu command không nhận (pip install), thêm PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## Kết nối SSH đầu tiên & Inventory cơ bản

Tạo folder dự án:

```bash
mkdir chomusuke-vps-ansible
cd chomusuke-vps-ansible
```

Tạo file inventory (YAML dễ đọc hơn ini):

```yml
# inventories/production/hosts.yml
all:
  hosts:
    my-vps:
      ansible_host: YOUR_VPS_IP_HERE          # ví dụ: 123.45.67.89
      ansible_user: root                      # hoặc ubuntu / user của bạn
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

Test kết nối (module ping – lệnh đơn giản nhất):

```bash
ansible -i inventories/production/hosts.yml my-vps -m ping
```

Thành công:

```bash
my-vps | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Nếu lỗi:

* Kiểm tra SSH key (`ssh-copy-id` trước).
* Thử thêm `-k` để hỏi password tạm thời: `ansible ... -k`

#### Ad-hoc command đầu tiên: Chạy lệnh thật trên VPS

Kiểm tra uptime:

```bash
ansible my-vps -i inventories/production/hosts.yml -m command -a "uptime"
```

Update package (an toàn, idempotent):

{% code overflow="wrap" %}
```bash
ansible my-vps -i inventories/production/hosts.yml -m apt -a "update_cache=yes upgrade=yes" --become
```
{% endcode %}

`--become` = chạy với sudo.

Chúc mừng! Bạn đã chạy thành công Ansible đầu tiên.
