# Arch Linux Server Setup

## Variables

```sh
HOSTNAME="archlinux"

IFACE="enp0s3"
IPADDR="192.168.1.100"
PREFIXLEN="24"
GATEWAY="192.168.1.1"
DNS="8.8.8.8"
```

## Set Hostname

```sh
echo "${HOSTNAME}" > /etc/hostname
```

Configure local hostname resolution:

```sh
cat > /etc/hosts << EOF
127.0.0.1 localhost
::1 localhost
127.0.1.1 ${HOSTNAME}.localdomain ${HOSTNAME}
EOF
```

## Enable Networking

Enable required services:

```sh
systemctl enable systemd-networkd --now
systemctl enable systemd-resolved --now
```

Configure a static IP:

```sh
printf "[Match]\nName=%s\n\n[Network]\nAddress=%s/%s\nGateway=%s\nDNS=%s\n" \
  "${IFACE}" "${IPADDR}" "${PREFIXLEN}" "${GATEWAY}" "${DNS}" \
  | tee /etc/systemd/network/10-static.network
```

Restart networking:

```sh
systemctl restart systemd-networkd
```

Disable conflicting network managers if present:

```sh
systemctl disable NetworkManager --now
```

Configure DNS resolver:

```sh
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Verify:

```sh
networkctl status ${IFACE}
ip addr show ${IFACE}
ip route
resolvectl status
ping -c 4 archlinux.org
```

## Create Administrative User

Install sudo:

```sh
pacman -S sudo
```

Create a regular user:

```sh
useradd -m -G wheel username
passwd username
```

Allow wheel group members to use sudo:

```sh
EDITOR=nano visudo
```

Uncomment:

```text
%wheel ALL=(ALL:ALL) ALL
```

## Configure SSH

Install OpenSSH:

```sh
pacman -S openssh
```

Enable the service:

```sh
systemctl enable --now sshd
```

Verify:

```sh
systemctl status sshd
ss -tlnp | grep ssh
```

(Optional) Harden SSH:

```sh
nano /etc/ssh/sshd_config
```

Recommended settings:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```sh
systemctl restart sshd
```

## Configure Firewall

Install UFW:

```sh
pacman -S ufw
```

Enable the service:

```sh
systemctl enable --now ufw
```

Set default policies:

```sh
ufw default deny incoming
ufw default allow outgoing
```

Allow SSH:

```sh
ufw allow ssh
```

Enable the firewall:

```sh
ufw enable
```

Verify status:

```sh
ufw status verbose
```

Examples:

```sh
# HTTP
ufw allow 80/tcp

# HTTPS
ufw allow 443/tcp

# Custom port
ufw allow 8080/tcp

# Remove a rule
ufw delete allow 8080/tcp
```

## Configure Automatic Maintenance

Update packages:

```sh
pacman -Syu
```

Clean old package cache:

```sh
pacman -Sc
```

## Verify Services

Check for failed services:

```sh
systemctl --failed
```

Check enabled services:

```sh
systemctl list-unit-files --state=enabled
```

Check listening ports:

```sh
ss -tulpn
```

Verify storage:

```sh
lsblk
df -h
```

Verify logs:

```sh
journalctl -p 3 -xb
```

## Reboot and Test

```sh
reboot
```

After reboot verify:

```sh
ping archlinux.org
ssh username@server-ip
systemctl --failed
ufw status
```