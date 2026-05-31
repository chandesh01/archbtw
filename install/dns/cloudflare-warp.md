# Cloudflare WARP on Arch Linux
https://developers.cloudflare.com/warp-client/get-started/linux/

Cloudflare WARP provides encrypted DNS and optional VPN-style traffic tunneling through Cloudflare's network.

WARP supports:

* DNS-over-HTTPS (DoH)
* Encrypted DNS queries
* Malware filtering with 1.1.1.1 for Families
* Full-device tunneling
* WireGuard and MASQUE protocols

## Install Cloudflare WARP

Install the package from the AUR.

First install build tools:

```sh
pacman -S --needed git base-devel
```

Clone the package:

```sh
git clone https://aur.archlinux.org/cloudflare-warp-bin.git
cd cloudflare-warp-bin
```

Build and install:

```sh
makepkg -si
```

Verify installation:

```sh
warp-cli --version
```

## Register the Client

Before connecting, register the device:

```sh
warp-cli registration new
```

Verify registration:

```sh
warp-cli registration show
```

## Connect to WARP

Connect:

```sh
warp-cli connect
```

Check status:

```sh
warp-cli status
```

Verify connectivity:

```sh
curl https://www.cloudflare.com/cdn-cgi/trace
```

Expected output:

```text
warp=on
```

## DNS-Only Mode

Use encrypted DNS without routing all traffic through Cloudflare:

```sh
warp-cli mode doh
warp-cli connect
```

Verify:

```sh
warp-cli status
```

This mode is similar to using a secure DNS provider while leaving network traffic unchanged.

## Full WARP Mode

Route traffic through Cloudflare's network:

```sh
warp-cli mode warp+doh
warp-cli connect
```

Verify:

```sh
warp-cli status
```

## Configure Protocol

Cloudflare WARP supports two tunnel protocols.

View current protocol:

```sh
warp-cli tunnel protocol get
```

### MASQUE (Default)

Recommended:

```sh
warp-cli tunnel protocol set MASQUE
```

### WireGuard

Alternative protocol:

```sh
warp-cli tunnel protocol set WireGuard
```

Reconnect after changing:

```sh
warp-cli disconnect
warp-cli connect
```

## Enable Family Filtering

### Disable Filtering

```sh
warp-cli dns families off
```

### Malware Protection

```sh
warp-cli dns families malware
```

### Malware + Adult Content Filtering

```sh
warp-cli dns families full
```

Verify:

```sh
warp-cli settings
```

## Enable WARP+ Unlimited

If you have an active WARP+ subscription:

On a mobile device:

```text
1.1.1.1 App
  → Settings
  → Account
  → Copy License Key
```

Register the license:

```sh
warp-cli registration license YOUR_LICENSE_KEY
```

Verify:

```sh
warp-cli registration show
```

Expected:

```text
Account type: Unlimited
```

## Useful Commands

Connect:

```sh
warp-cli connect
```

Disconnect:

```sh
warp-cli disconnect
```

Status:

```sh
warp-cli status
```

Settings:

```sh
warp-cli settings
```

Show registration:

```sh
warp-cli registration show
```

Help:

```sh
warp-cli --help
```

## Troubleshooting

Check service status:

```sh
systemctl status warp-svc
```

Restart service:

```sh
sudo systemctl restart warp-svc
```

Generate diagnostics:

```sh
sudo warp-diag
```

Submit diagnostics to Cloudflare:

```sh
sudo warp-diag feedback
```

## Uninstall

Remove the package:

```sh
sudo pacman -Rns cloudflare-warp-bin
```

Remove registration:

```sh
warp-cli registration delete
```

## Comparison

| Solution   | Encryption | Filtering | Recursive DNS | VPN |
| ---------- | ---------- | --------- | ------------- | --- |
| WARP (DoH) | Yes        | Optional  | No            | No  |
| WARP + DoH | Yes        | Optional  | No            | Yes |
| NextDNS    | Yes        | Yes       | No            | No  |

For most desktop users:

* **NextDNS** → Best DNS filtering solution.
* **Cloudflare WARP** → Best Cloudflare-integrated encrypted DNS and VPN solution.