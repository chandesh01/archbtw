# DNS Configuration with systemd-resolved

`systemd-resolved` provides:

* Local DNS caching
* DNSSEC validation
* DNS-over-TLS (DoT)
* Split DNS support
* Integration with NetworkManager and systemd-networkd

The local resolver listens on:

```text
127.0.0.53
```

## Configure resolv.conf

### Installed System

Configure the systemd-resolved stub resolver:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Verify:

```sh
cat /etc/resolv.conf
```

Expected:

```text
nameserver 127.0.0.53
```

### During Installation (Before Chroot)

If configuring the installed system from the Arch ISO:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
```

This ensures the installed system uses the systemd-resolved stub resolver after boot.

## Optional: Enable DNS-over-TLS and DNSSEC

Edit:

```sh
nano /etc/systemd/resolved.conf
```

Example:

```ini
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
FallbackDNS=9.9.9.9#dns.quad9.net

DNSOverTLS=yes
DNSSEC=allow-downgrade
```

Options:

* `DNSOverTLS=yes` enables encrypted DNS queries.
* `DNSSEC=allow-downgrade` validates DNSSEC when supported.
* `DNSSEC=yes` enforces DNSSEC validation and may break resolution on misconfigured networks.

## Enable Service

Enable and start the resolver:

```sh
systemctl enable --now systemd-resolved
```

Verify:

```sh
systemctl status systemd-resolved
```

## Verify Configuration

Check resolver status:

```sh
resolvectl status
```

Test DNS resolution:

```sh
resolvectl query archlinux.org
```

View active DNS servers:

```sh
resolvectl dns
```

Flush caches:

```sh
resolvectl flush-caches
```

Show statistics:

```sh
resolvectl statistics
```

## NetworkManager Integration

Configure NetworkManager to use systemd-resolved:

```sh
mkdir -p /etc/NetworkManager/conf.d

cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=systemd-resolved
EOF
```

Restart:

```sh
systemctl restart NetworkManager
```

Verify:

```sh
resolvectl status
```

## Troubleshooting

Check which service is listening on port 53:

```sh
ss -tulpn | grep :53
```

Check resolver logs:

```sh
journalctl -u systemd-resolved
```

Restart the resolver:

```sh
systemctl restart systemd-resolved
```

Restore the default stub resolver:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## Notes

* Recommended for modern Arch Linux installations.
* Integrates cleanly with NetworkManager and systemd-networkd.
* Works well with Cloudflare, Quad9, NextDNS, and local DNS servers.
* Required by many advanced DNS setups documented elsewhere in this guide.

## References

* ArchWiki: systemd-resolved
* systemd-resolved(8)
* resolvectl(1)
