# Unbound + AdGuard DNS Proxy

This setup combines:

* **Unbound** for recursive DNS resolution and DNSSEC validation
* **AdGuard DNS Proxy** (`dnsproxy`) for DNS-over-HTTPS (DoH), DNS-over-TLS (DoT), and DNS-over-QUIC (DoQ)
* Optional integration with **systemd-resolved** and **NetworkManager**

## DNS Flow

### Desktop / Laptop

```text
Applications
      ↓
systemd-resolved
      ↓
dnsproxy
      ↓
Unbound
      ↓
DNS Root Servers
```

### Standalone Server

```text
Applications
      ↓
dnsproxy
      ↓
Unbound
      ↓
DNS Root Servers
```

## Benefits

* Local recursive DNS
* DNSSEC validation
* Local DNS caching
* Encrypted DNS transport
* NetworkManager integration
* Optional standalone operation

## Install Packages

```sh
pacman -S unbound adguard-dnsproxy
```

## Configure Unbound

Configure Unbound to listen on port `5335`.

Create:

```sh
mkdir -p /etc/unbound/unbound.conf.d
nano /etc/unbound/unbound.conf.d/local.conf
```

Example configuration:

```conf
server:
    interface: 127.0.0.1
    port: 5335

    access-control: 127.0.0.0/8 allow

    prefetch: yes
    qname-minimisation: yes

    auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

Initialize DNSSEC trust anchors:

```sh
unbound-anchor -a /var/lib/unbound/root.key
```

Enable Unbound:

```sh
systemctl enable --now unbound
```

Verify:

```sh
ss -tulpn | grep 5335
```

Expected:

```text
127.0.0.1:5335
```

## Configure AdGuard DNS Proxy

Create:

```sh
nano /etc/systemd/system/dnsproxy.service
```

```ini
[Unit]
Description=AdGuard DNS Proxy
After=network-online.target

[Service]
EnvironmentFile=-/etc/default/dnsproxy
ExecStart=/usr/bin/dnsproxy $DNSPROXY_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```sh
systemctl daemon-reload
```

# Method A: Standalone DNS Stack

Recommended for:

* Servers
* Homelabs
* Self-hosted infrastructure

Configure dnsproxy:

```sh
cat > /etc/default/dnsproxy << EOF
DNSPROXY_OPTS="--listen=127.0.0.1:53 --upstream=127.0.0.1:5335 --cache"
EOF
```

Disable conflicting services:

```sh
systemctl disable --now systemd-resolved dnsmasq
```

Configure local DNS:

```sh
rm -f /etc/resolv.conf
echo "nameserver 127.0.0.1" > /etc/resolv.conf
```

Optional:

```sh
chattr +i /etc/resolv.conf
```

Enable services:

```sh
systemctl enable --now unbound
systemctl enable --now dnsproxy
```

Verify:

```sh
dig @127.0.0.1 archlinux.org
```

# Method B: NetworkManager + systemd-resolved

Recommended for:

* Desktop systems
* Laptops
* Workstations

Configure dnsproxy:

```sh
cat > /etc/default/dnsproxy << EOF
DNSPROXY_OPTS="--listen=127.0.0.1:5053 --upstream=127.0.0.1:5335 --cache"
EOF
```

Configure NetworkManager:

```sh
mkdir -p /etc/NetworkManager/conf.d

cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=systemd-resolved
EOF
```

Configure systemd-resolved:

```sh
mkdir -p /etc/systemd/resolved.conf.d

cat > /etc/systemd/resolved.conf.d/dnsproxy.conf << EOF
[Resolve]
DNS=127.0.0.1:5053
DNSSEC=yes
EOF
```

Use the stub resolver:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Enable services:

```sh
systemctl enable --now systemd-resolved
systemctl enable --now dnsproxy

systemctl restart systemd-resolved
systemctl restart NetworkManager
```

## Verify

Check services:

```sh
systemctl status unbound
systemctl status dnsproxy
systemctl status systemd-resolved
```

Verify DNS configuration:

```sh
resolvectl status
```

Expected DNS server:

```text
127.0.0.1:5053
```

Query dnsproxy:

```sh
dig @127.0.0.1 -p 5053 archlinux.org
```

Query Unbound directly:

```sh
dig @127.0.0.1 -p 5335 archlinux.org
```

Test DNSSEC validation:

```sh
dig +dnssec dnssec-failed.org @127.0.0.1 -p 5335
```

The lookup should fail validation.

## Optional: Forward Unbound to Encrypted Upstreams

Instead of recursive resolution, Unbound can forward queries to DNS-over-TLS providers.

Example:

```conf
forward-zone:
    name: "."
    forward-tls-upstream: yes

    forward-addr: 1.1.1.1@853
    forward-addr: 1.0.0.1@853
```

Restart:

```sh
systemctl restart unbound
```

## Restore Defaults

Remove resolver override:

```sh
rm -f /etc/systemd/resolved.conf.d/dnsproxy.conf
```

Restore resolver:

```sh
chattr -i /etc/resolv.conf 2>/dev/null
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Disable dnsproxy:

```sh
systemctl disable --now dnsproxy
```

Restart services:

```sh
systemctl restart systemd-resolved
systemctl restart NetworkManager
```

## Summary

| Component        | Role                        | Port                 |
| ---------------- | --------------------------- | -------------------- |
| Unbound          | Recursive resolver + DNSSEC | 127.0.0.1:5335       |
| dnsproxy         | DoH / DoT / DoQ proxy       | 127.0.0.1:53 or 5053 |
| systemd-resolved | System resolver             | 127.0.0.53           |

## Should You Use This?

| Setup              | Complexity | Privacy   | Filtering | Self Hosted |
| ------------------ | ---------- | --------- | --------- | ----------- |
| NextDNS            | Low        | Good      | Excellent | No          |
| Cloudflare WARP    | Low        | Good      | Basic     | No          |
| Unbound            | Medium     | Excellent | None      | Yes         |
| Unbound + dnsproxy | High       | Excellent | Optional  | Yes         |

### Recommendation

* **Desktop/Laptop:** Use Method B with NetworkManager and systemd-resolved.
* **Server:** Use Method A for a standalone DNS stack.
* **Most users:** NextDNS is simpler and easier to maintain.
* **Maximum control:** Use Unbound recursion with dnsproxy and DNSSEC validation.
