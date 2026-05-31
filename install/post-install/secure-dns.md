# Secure DNS with NextDNS

Configure NextDNS with NetworkManager and systemd-resolved to provide:

* DNS-over-HTTPS (DoH)
* Ad and tracker blocking
* Custom DNS policies
* Split-horizon DNS forwarding
* Encrypted upstream DNS servers

## Prerequisites

This guide assumes the system is already configured with:

* NetworkManager
* systemd-resolved

Verify:

```sh
systemctl is-enabled NetworkManager
systemctl is-enabled systemd-resolved
```

## Install NextDNS

Install the NextDNS client from the AUR:

```sh
paru -S nextdns
```

or:

```sh
yay -S nextdns
```

Enable the service:

```sh
systemctl enable --now nextdns
```

Verify:

```sh
systemctl status nextdns
```

## Configure systemd-resolved

Ensure `/etc/resolv.conf` points to the systemd-resolved stub resolver:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Verify:

```sh
cat /etc/resolv.conf
```

Expected output:

```text
nameserver 127.0.0.53
```

## Configure NetworkManager

Prevent DHCP from overriding local DNS settings.

List available connections:

```sh
nmcli connection show
```

Example:

```text
Wired connection 1
```

Configure NetworkManager to use the local resolver:

```sh
nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
nmcli connection modify "Wired connection 1" ipv4.dns "127.0.0.1"

nmcli connection modify "Wired connection 1" ipv6.ignore-auto-dns yes
nmcli connection modify "Wired connection 1" ipv6.dns "::1"
```

Reconnect:

```sh
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"
```

## Configure NextDNS

Obtain your configuration ID from the NextDNS dashboard and configure the client:

```sh
nextdns config set -config YOUR_CONFIG_ID
```

Activate the configuration:

```sh
nextdns activate
```

Restart services:

```sh
systemctl restart nextdns
systemctl restart systemd-resolved
```

## Configure Custom DoH Upstreams

Use Cloudflare as the upstream DNS provider:

```sh
nextdns config set -forwarder .=https://cloudflare-dns.com/dns-query#1.1.1.1,1.0.0.1
```

Restart NextDNS:

```sh
nextdns restart
```

You may replace Cloudflare with any DNS-over-HTTPS provider.

> **Note:** NextDNS already provides encrypted DNS. Configuring a custom DoH upstream is optional and only needed when you want NextDNS to forward queries to another DNS provider instead of using the default NextDNS infrastructure.

## Configure Split-Horizon DNS

Forward selected domains to specific DNS servers.

Example:

```sh
nextdns config set \
  -forwarder internal.example.com=192.168.1.1 \
  -forwarder corp.example.com=https://dns.example.com/dns-query#10.0.0.1
```

This configuration routes:

* `internal.example.com` through a local DNS server
* `corp.example.com` through a custom DoH resolver

## Verify Configuration

Check resolver status:

```sh
resolvectl status
```

The active DNS server should be:

```text
127.0.0.1
```

Install DNS utilities:

```sh
pacman -S bind
```

Query through NextDNS:

```sh
dig @127.0.0.1 google.com
```

Expected output:

```text
SERVER: 127.0.0.1#53
```

Verify encrypted upstream connections:

```sh
ss -tupn | grep nextdns
```

You should see active connections to remote HTTPS endpoints on port `443`.

## Troubleshooting

Restart all DNS-related services:

```sh
systemctl restart NetworkManager
systemctl restart systemd-resolved
systemctl restart nextdns
```

Flush resolver caches:

```sh
resolvectl flush-caches
```

View NextDNS logs:

```sh
journalctl -u nextdns -f
```

View resolver status:

```sh
resolvectl status
```


