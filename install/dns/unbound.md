# Unbound DNS Resolver

Unbound is a validating, caching, recursive DNS resolver.

Instead of sending DNS queries to public resolvers such as Cloudflare, Google, or NextDNS, Unbound can query the DNS hierarchy directly and validate responses using DNSSEC.

## Why Use Unbound?

Benefits:

* Local DNS cache
* Recursive DNS resolution
* DNSSEC validation
* Reduced reliance on third-party DNS providers
* Local DNS overrides and split-horizon DNS
* Popular in homelabs and self-hosted environments

## DNS Flow

Typical DNS:

```text
Applications
      ↓
systemd-resolved
      ↓
Cloudflare / Google / NextDNS
```

With Unbound:

```text
Applications
      ↓
systemd-resolved
      ↓
Unbound
      ↓
DNS Root Servers
      ↓
Authoritative DNS Servers
```

## Unbound vs NextDNS

| Feature           | Unbound | NextDNS |
| ----------------- | ------- | ------- |
| Recursive DNS     | Yes     | No      |
| DNSSEC Validation | Yes     | Yes     |
| Ad Blocking       | No      | Yes     |
| Analytics         | No      | Yes     |
| Self Hosted       | Yes     |         |
| Easy Setup        | Medium  | Easy    |

Use **NextDNS** for filtering and simplicity.

Use **Unbound** if you want a self-hosted recursive resolver.

---

# Setup Unbound

## Prerequisites

Configure:

* systemd-resolved
* NetworkManager (optional)

See:

```text
DNS Configuration with systemd-resolved
```

---

## Install

```sh
pacman -S unbound
```

## Generate Root Trust Anchor

Required for DNSSEC validation:

```sh
unbound-anchor -a /etc/unbound/root.key
```

## Configure Unbound

Create:

```sh
mkdir -p /etc/unbound/unbound.conf.d
nano /etc/unbound/unbound.conf.d/local.conf
```

Minimal recursive configuration:

```conf
server:
    interface: 127.0.0.1
    port: 5335

    do-ip4: yes
    do-ip6: yes

    prefetch: yes
    qname-minimisation: yes

    cache-min-ttl: 3600
    cache-max-ttl: 86400

    auto-trust-anchor-file: "/etc/unbound/root.key"
```

## Enable Service

```sh
systemctl enable --now unbound
```

Verify:

```sh
systemctl status unbound
```

Check listening port:

```sh
ss -tulpn | grep 5335
```

Expected:

```text
127.0.0.1:5335
```

---

## Configure systemd-resolved

Create:

```sh
mkdir -p /etc/systemd/resolved.conf.d
```

```sh
cat > /etc/systemd/resolved.conf.d/unbound.conf << EOF
[Resolve]
DNS=127.0.0.1:5335
EOF
```

Restart:

```sh
systemctl restart systemd-resolved
```

Ensure the resolver symlink exists:

```sh
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

---

## Verify

Check resolver status:

```sh
resolvectl status
```

Expected DNS server:

```text
127.0.0.1:5335
```

Install DNS utilities:

```sh
pacman -S bind
```

Query through Unbound:

```sh
dig @127.0.0.1 -p 5335 archlinux.org
```

Test DNSSEC:

```sh
dig @127.0.0.1 -p 5335 archlinux.org +dnssec
```

Test DNSSEC failure handling:

```sh
dig @127.0.0.1 -p 5335 dnssec-failed.org
```

The query should fail validation.

---

## Optional: Forwarding Mode

Instead of full recursion, Unbound can forward queries to encrypted upstream resolvers.

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

---

## Optional: Local DNS Records

Example local overrides:

```conf
server:
    local-zone: "home.arpa." static
    local-data: "nas.home.arpa. IN A 192.168.1.10"
```

Restart:

```sh
systemctl restart unbound
```

Verify:

```sh
ping nas.home.arpa
```

---

## Troubleshooting

Check service status:

```sh
systemctl status unbound
```

View logs:

```sh
journalctl -u unbound
```

Verify resolver configuration:

```sh
resolvectl status
```

Check port usage:

```sh
ss -tulpn | grep 5335
```

Test resolution directly:

```sh
dig @127.0.0.1 -p 5335 archlinux.org
```

---

## Recommendation

For Arch Linux systems using NetworkManager and systemd-resolved, run Unbound on **127.0.0.1:5335** and leave the systemd-resolved stub listener on **127.0.0.53**. This avoids port conflicts and integrates cleanly with the default networking stack.
