---
icon: key
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/fZMM9Pd2vdURETYcbaWM/vps/getting-started/first-time-ssh-into-vps-as-root
---

# First Time SSH into VPS as Root

When you first receive a VPS from a provider (DigitalOcean, Vultr, Linode, etc.), the initial step is to connect to the server via SSH using the **root** account. This is the only secure and effective way to manage the server remotely.

This guide provides detailed instructions for the first SSH connection on **macOS** and **Linux** (Ubuntu/Debian desktop), while focusing on the server operating system **Ubuntu/Debian**.

## Information You Need to Prepare

From your VPS provider, you will receive:
* **VPS IP address** (example: `123.45.67.89`)
* **Login account**: `root`
* **Root password** (sent via email or displayed on the dashboard)
* **SSH port**: Default is `22` (the provider will clearly notify you if it has been changed)

{% hint style="info" %}
**Important security note**: Logging in directly as root with a password should only be done **the first time**. Immediately after successfully connecting, you should perform basic security steps (create a regular user with sudo privileges, set up SSH keys, disable root login, etc.). These steps will be covered in detail in subsequent articles.
{% endhint %}

## Connecting via SSH on macOS and Linux

Both macOS and Linux come with a built-in Terminal, making the connection process very straightforward.

{% stepper %}
{% step %}
Open **Terminal**:
* macOS: Press `Cmd + Space` → type "Terminal"
* Linux (Ubuntu/Debian): Press `Ctrl + Alt + T`
{% endstep %}
{% step %}
Enter the connection command:

```bash
ssh root@123.45.67.89
```

(Replace `123.45.67.89` with your actual VPS IP)

{% endstep %}

{% step %}

On the first connection, the system will ask to confirm the fingerprint:

```bash
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

→ Type yes and press Enter.
{% endstep %}

{% step %}
Enter the root password:

* While typing the password, no characters will be displayed (not even dots or asterisks). This is a normal security feature.
* After typing, press Enter.
{% endstep %}
{% endstepper %}

If successful, you will see a prompt similar to:

```bash
root@hostname:~#
```

**If the SSH port is not 22** (some providers change the port for added security):

```bash
ssh root@123.45.67.89 -p 2222
```

(Replace `2222` with the actual port)

## Common Errors and Solutions

* **Connection timed out** or **No route to host**: Double-check the IP/port, ensure the VPS is running, and your internet connection is stable. The provider's firewall may not have opened the SSH port yet.
* **Permission denied (publickey, password)**: Incorrect password or the provider has disabled password login. Try carefully copying and pasting the password, or contact support for verification.
* **Connection refused**: Wrong SSH port or the SSH service is not running (rare with new VPS).

If you enter the wrong password multiple times and get temporarily locked out, contact the provider's support to reset it or use the emergency console (VNC/KVM) if available.
