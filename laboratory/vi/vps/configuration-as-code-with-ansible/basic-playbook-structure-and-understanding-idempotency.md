---
icon: memo-circle-check
---

# Cấu trúc Playbook Cơ Bản & Idempotency

**Playbook** là file YAML chứa các lệnh (tasks) mà Ansible sẽ thực hiện theo thứ tự. Nó giống như một "script" nhưng declarative (bạn nói "tôi muốn server như thế này", Ansible tự lo phần còn lại).

## Tại sao playbook mạnh hơn ad-hoc command?

* Dễ tái sử dụng: Viết một lần, chạy mãi mãi.
* Idempotent: Chạy nhiều lần → kết quả giống nhau (không làm hỏng nếu đã đúng).
* Dễ đọc, dễ version control (Git).
* Có thể tổ chức thành nhiều playbook, role sau này.

## Cấu trúc cơ bản của một playbook

Tạo file đầu tiên trong repo của bạn: `playbooks/first-playbook.yml`

```yaml
---
- name: My first Ansible playbook
  hosts: columbina               # hoặc all, webservers... (tên host/group từ inventory)
  become: true                   # Chạy với sudo (tương đương --become)
  gather_facts: true             # Thu thập thông tin server (facts)

  tasks:
    - name: Ensure the system is up to date
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install essential packages
      apt:
        name:
          - git
          - curl
          - unzip
          - vim
        state: present

    - name: Create a simple text file
      copy:
        content: "Hello from Ansible!\nWelcome to Chomusuke VPS Automation."
        dest: /tmp/hello-ansible.txt
        mode: '0644'
```

Giải thích từng phần:

* `---` : Bắt đầu file YAML.
* `- name:` : Tên playbook (hiển thị khi chạy).
* `hosts:` : Đối tượng thực hiện (từ inventory: columbina, all...).
* `become: true` : Chạy với quyền root (sudo).
* `gather_facts: true` : Ansible tự thu thập thông tin server (OS, CPU, memory...).
* `tasks:` : Danh sách các công việc (task). Mỗi task có:
  * `name`: Mô tả task (hiển thị khi chạy).
  * Module (apt, copy, file, service...) + tham số.

## Chạy playbook đầu tiên

Di chuyển vào thư mục repo:

```bash
cd chomusuke-vps-ansible
```

Chạy với dry-run (test không thay đổi thật):

{% code overflow="wrap" %}
```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/first-playbook.yml --check --diff
```
{% endcode %}

Nếu ok (không lỗi syntax), chạy thật:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/first-playbook.yml
```

Quan sát output:

* Mỗi task sẽ báo **ok** (đã đạt trạng thái), **changed** (đã thay đổi), hoặc **failed**.
* Chạy lần 2: Hầu hết sẽ là **ok** (changed: false) → chứng minh idempotent!

## Minh họa idempotency

Hãy thử chạy playbook trên 2-3 lần:

* Lần 1: Có thể changed=true (cài package, tạo file).
* Lần 2: changed=false hết (vì package đã cài, file đã tồn tại đúng nội dung).

→ Đây là điểm mạnh nhất của Ansible so với Bash script (script Bash thường chạy lại sẽ lặp lại lệnh, có thể gây lỗi).

## Một số mẹo khi viết playbook

* Luôn thêm `name:` cho task → dễ debug.
* Dùng `--check --diff` trước khi apply thật.
* Nếu cần hỏi sudo password: thêm `--ask-become-pass`.
* Để verbose (xem chi tiết): `-v`, `-vv`, `-vvv`.
* Tạo file `ansible.cfg` ở root repo để cấu hình mặc định (sẽ nói chi tiết bài sau).

Ví dụ ansible.cfg cơ bản (tạo file ansible.cfg):

```ini
[defaults]
inventory = inventories/production/hosts.yml
host_key_checking = False          # Bỏ kiểm tra SSH key lần đầu (test nhanh)
interpreter_python = auto_silent   # Giảm warning interpreter
```

## Bài tập nhỏ để thực hành

1.  Thêm task mới vào playbook: Tạo user non-root (nếu chưa có).<br>

    ```yml
    - name: Create non-root user
      user:
        name: reishou
        shell: /bin/bash
        groups: sudo
        append: yes
        create_home: yes
    ```


2. Chạy playbook với `--check` trước.
3. Chạy thật → kiểm tra SSH vào VPS với user mới (ssh reishou@IP).

Chúc mừng! Bạn đã viết và chạy playbook đầu tiên.
