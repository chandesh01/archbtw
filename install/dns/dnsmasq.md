# dnsmasq DNS Cache and DNSSEC

`dnsmasq` is a lightweight DNS forwarder and cache.

Common uses:

* DNS caching
* DNSSEC validation
* Local DNS overrides
* Custom DNS forwarding
* Homelabs and self-hosted environments

## Deployment Modes

### NetworkManager Managed

Recommended for:

* Desktop systems
* Laptops
* Typical Arch installations

```text
Applications
      ↓
NetworkManager
      ↓
dnsmasq
      ↓
Upstream DNS
```

### Standalone

Recommended for:

* Servers
* Homelabs
* Advanced DNS configurations

```text
Applications
      ↓
dnsmasq
      ↓
Upstream DNS
```

# Method 1: NetworkManager + dnsmasq

## Install

```sh
pacman -S dnsmasq
```

## Configure NetworkManager

```sh
mkdir -p /etc/NetworkManager/conf.d

cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=dnsmasq
EOF
```

Restart:

```sh
systemctl restart NetworkManager
```

## Enable DNSSEC

```sh
mkdir -p /etc/NetworkManager/dnsmasq.d

cat > /etc/NetworkManager/dnsmasq.d/dnssec.conf << EOF
conf-file=/usr/share/dnsmasq/trust-anchors.conf
dnssec
EOF
```

Reload:

```sh
systemctl restart NetworkManager
```

## Optional Custom Configuration

Create:

```sh
nano /etc/NetworkManager/dnsmasq.d/custom.conf
```

Example:

```ini
server=1.1.1.1
address=/nas.home.arpa/192.168.1.10
```

## Verify

Check resolver:

```sh
resolvectl status
```

View resolver configuration:

```sh
cat /etc/resolv.conf
```

Test DNSSEC:

```sh
pacman -S bind

dig +dnssec google.com
dig +dnssec dnssec-failed.org
```

# Method 2: Standalone dnsmasq

## Install

```sh
pacman -S dnsmasq
```

## Disable systemd-resolved

```sh
systemctl disable --now systemd-resolved
```

Remove existing resolver configuration:

```sh
rm -f /etc/resolv.conf
```

Create a local resolver:

```sh
echo "nameserver 127.0.0.1" > /etc/resolv.conf
```

## Configure dnsmasq

Edit:

```sh
nano /etc/dnsmasq.conf
```

### DNSSEC Validation

```ini
no-resolv

listen-address=127.0.0.1,::1

dnssec
conf-file=/usr/share/dnsmasq/trust-anchors.conf

server=1.1.1.1
server=1.0.0.1
```

### DNSSEC Proxy Mode

If upstream DNS already validates DNSSEC:

```ini
no-resolv

listen-address=127.0.0.1,::1

proxy-dnssec

server=1.1.1.1
server=1.0.0.1
```

Use either `dnssec` or `proxy-dnssec`, not both.

## Prevent NetworkManager DNS Changes

Create:

```sh
mkdir -p /etc/NetworkManager/conf.d
```

```sh
cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=none
EOF
```

Restart:

```sh
systemctl restart NetworkManager
```

## Start dnsmasq

```sh
systemctl enable --now dnsmasq
```

Verify:

```sh
ss -tulpn | grep :53
```

Expected:

```text
127.0.0.1:53
::1:53
```

## Test DNSSEC

Valid domain:

```sh
dig +dnssec @127.0.0.1 google.com
```

Invalid domain:

```sh
dig +dnssec @127.0.0.1 dnssec-failed.org
```

The invalid domain should fail validation.

# Local DNS Records

Create:

```sh
nano /etc/dnsmasq.d/local.conf
```

Example:

```ini
address=/nas.home.arpa/192.168.1.10
address=/router.home.arpa/192.168.1.1
```

Restart:

```sh
systemctl restart dnsmasq
```

Verify:

```sh
dig nas.home.arpa
```

# Troubleshooting

Check service:

```sh
systemctl status dnsmasq
```

Check port usage:

```sh
ss -tulpn | grep :53
```

Check resolver:

```sh
resolvectl status
```

View logs:

```sh
journalctl -u dnsmasq -f
```

# Comparison

| Feature             | NM + dnsmasq | Standalone dnsmasq |
| ------------------- | ------------ | ------------------ |
| Easy Setup          | Yes          | Moderate           |
| DNSSEC              | Yes          | Yes                |
| Local DNS Overrides | Yes          | Yes                |
| Full DNS Control    | Limited      | Yes                |
| Server Friendly     | No           | Yes                |
| Desktop Friendly    | Yes          | Yes                |

## Recommendation

* **Desktop/Laptop:** Use NetworkManager-managed dnsmasq.
* **Server:** Use standalone dnsmasq.
* **Maximum control:** Use standalone dnsmasq with DNSSEC validation.
