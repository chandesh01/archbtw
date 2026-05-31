# Secure Boot on Arch Linux using Shim + MOK + systemd-boot + UKI

## Boot Flow

```text
UEFI Firmware
   ↓ trusts Microsoft keys
shimx64.efi
   ↓ trusts your MOK key
grubx64.efi
   ↓ actually signed systemd-boot
Unified Kernel Image (UKI)
   ↓
Linux boots
```

This allows Secure Boot without replacing firmware keys.

## Install Required Packages

Install Secure Boot tools:

```sh
sudo pacman -S sbsigntools mokutil
```

Install `shim-signed` from AUR:

```sh
git clone https://aur.archlinux.org/shim-signed.git
cd shim-signed
makepkg -si
```

Install systemd-boot:

```sh
sudo bootctl install
```

## Verify Secure Boot State

Check Secure Boot status:

```sh
mokutil --sb-state
```

Expected:

```text
SecureBoot enabled
```

## Generate Machine Owner Keys (MOK)

Generate signing keys:

```sh
openssl req -newkey rsa:2048 -nodes -keyout MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=Machine Owner Key/" -out MOK.crt
```

Convert certificate to DER format:

```sh
openssl x509 -outform DER -in MOK.crt -out MOK.cer
```

Generated files:

| File      | Purpose                        |
| --------- | ------------------------------ |
| `MOK.key` | Private signing key            |
| `MOK.crt` | PEM certificate                |
| `MOK.cer` | DER certificate for MokManager |

Use:

```text
RSA 2048
```

Some shim versions may fail with RSA 4096 keys.

## Store Keys Securely

Recommended location:

```sh
sudo mkdir -p /root/secureboot
sudo mv MOK.key MOK.crt MOK.cer /root/secureboot/
```

Set permissions:

```sh
sudo chmod 700 /root/secureboot
sudo chmod 600 /root/secureboot/*
```

## Copy Shim Files

Copy shim as fallback bootloader:

```sh
sudo cp /usr/share/shim-signed/shimx64.efi /efi/EFI/BOOT/BOOTX64.EFI
```

Copy MokManager:

```sh
sudo cp /usr/share/shim-signed/mmx64.efi /efi/EFI/BOOT/
```

## Sign systemd-boot

Sign the real systemd-boot EFI binary:

```sh
sudo sbsign --key /root/secureboot/MOK.key --cert /root/secureboot/MOK.crt --output /efi/EFI/systemd/systemd-bootx64.efi.signed /efi/EFI/systemd/systemd-bootx64.efi
```

Replace original:

```sh
sudo mv /efi/EFI/systemd/systemd-bootx64.efi.signed /efi/EFI/systemd/systemd-bootx64.efi
```

## Copy Signed systemd-boot as Shim Target

Shim expects a secondary EFI binary.

Use signed systemd-boot as:

```text
grubx64.efi
```

Copy signed binary:

```sh
sudo cp /efi/EFI/systemd/systemd-bootx64.efi /efi/EFI/BOOT/grubx64.efi
```

## Sign UKIs

#### Default UKI

```sh
sudo sbsign --key /root/secureboot/MOK.key --cert /root/secureboot/MOK.crt --output /efi/EFI/Linux/ArchLinux-linux.efi.signed /efi/EFI/Linux/ArchLinux-linux.efi
```

Replace original:

```sh
sudo mv /efi/EFI/Linux/ArchLinux-linux.efi.signed /efi/EFI/Linux/ArchLinux-linux.efi
```

#### Recovery UKI

```sh
sudo sbsign --key /root/secureboot/MOK.key --cert /root/secureboot/MOK.crt --output /efi/EFI/Linux/ArchLinux-recovery.efi.signed /efi/EFI/Linux/ArchLinux-recovery.efi
```

Replace original:

```sh
sudo mv /efi/EFI/Linux/ArchLinux-recovery.efi.signed /efi/EFI/Linux/ArchLinux-recovery.efi
```

## Verify Signatures

Verify systemd-boot:

```sh
sbverify --list /efi/EFI/systemd/systemd-bootx64.efi
```

Verify shim target:

```sh
sbverify --list /efi/EFI/BOOT/grubx64.efi
```

Verify UKIs:

```sh
sbverify --list /efi/EFI/Linux/ArchLinux-linux.efi
sbverify --list /efi/EFI/Linux/ArchLinux-recovery.efi
```

## Verify SBAT Section

Modern shim requires EFI binaries to contain:

```text
.sbat
```

Verify:

```sh
objdump -j .sbat -s /efi/EFI/systemd/systemd-bootx64.efi
```

and:

```sh
objdump -j .sbat -s /efi/EFI/Linux/ArchLinux-linux.efi
```

Modern `systemd-boot` and UKIs already contain SBAT metadata.

## Enroll MOK Certificate

Copy certificate to ESP:

```sh
sudo cp /root/secureboot/MOK.cer /efi/
```

Import certificate:

```sh
sudo mokutil --import /efi/MOK.cer
```

You will be prompted for a temporary password.

## Reboot and Enroll Key

Reboot.

Shim launches:

```text
MokManager
```

Choose:

1. `Enroll key from disk`
2. Select `MOK.cer`
3. Confirm enrollment
4. Enter password
5. Continue boot

Now shim trusts EFI binaries signed using your MOK.

## Create EFI Boot Entry

Create NVRAM entry:

```sh
sudo efibootmgr --unicode --disk /dev/nvme0n1 --part 1 --create --label "Shim" --loader '\EFI\BOOT\BOOTX64.EFI'
```

Adjust:

* disk
* partition
* ESP path

for your system.

## Verify Secure Boot

Check Secure Boot:

```sh
mokutil --sb-state
```

List enrolled keys:

```sh
mokutil --list-enrolled
```

## Automatic Re-Signing After Updates

Kernel updates regenerate UKIs.

`systemd` updates may replace:

```text
systemd-bootx64.efi
```

Create centralized signing script:

```sh
sudo nano /usr/local/bin/sign-efi
```

Contents:

```bash
##!/usr/bin/env bash
set -euo pipefail

KEY="/root/secureboot/MOK.key"
CERT="/root/secureboot/MOK.crt"

BOOT="/efi/EFI/systemd/systemd-bootx64.efi"
SHIM_TARGET="/efi/EFI/BOOT/grubx64.efi"

sign() {
    local file="$1"

    [[ -f "$file" ]] || return 0

    echo "Signing $file"

    sbsign \
        --key "$KEY" \
        --cert "$CERT" \
        --output "${file}.signed" \
        "$file"

    mv "${file}.signed" "$file"
}

## Sign systemd-boot
sign "$BOOT"

## Copy signed systemd-boot for shim
cp "$BOOT" "$SHIM_TARGET"

## Sign UKIs
find /efi/EFI/Linux -name "*.efi" -type f | while read -r file; do
    sign "$file"
done
```

Make executable:

```sh
sudo chmod +x /usr/local/bin/sign-efi
```

## Automatically Sign New UKIs

Create kernel-install hook:

```sh
sudo mkdir -p /etc/kernel/install.d
sudo nano /etc/kernel/install.d/99-sign-efi.install
```

Contents:

```bash
##!/usr/bin/env bash

case "$1" in
    add)
        /usr/local/bin/sign-efi
        ;;
esac
```

Make executable:

```sh
sudo chmod +x /etc/kernel/install.d/99-sign-efi.install
```

## Automatically Re-Sign After systemd Updates

Create pacman hook:

```sh
sudo mkdir -p /etc/pacman.d/hooks
sudo nano /etc/pacman.d/hooks/95-systemd-boot-sign.hook
```

Contents:

```ini
[Trigger]
Operation = Upgrade
Type = Package
Target = systemd

[Action]
Description = Re-signing systemd-boot for Secure Boot
When = PostTransaction
Exec = /usr/local/bin/sign-efi
```

## Optional: Disable Shim Validation

If Secure Boot is only needed for Windows requirements:

```sh
sudo mokutil --disable-validation
```

Then:

* Secure Boot remains enabled
* shim loads unsigned EFI binaries

## Kernel Module Warning

Unsigned kernel modules may fail to load under Secure Boot.

Especially:

* NVIDIA
* DKMS modules
* VirtualBox
* VMware
* ZFS

Such modules may require separate signing.

## Remove Shim Setup

Remove package:

```sh
sudo pacman -R shim-signed
```

Remove copied files:

```sh
sudo rm /efi/EFI/BOOT/BOOTX64.EFI
sudo rm /efi/EFI/BOOT/mmx64.efi
sudo rm /efi/EFI/BOOT/grubx64.efi
```

Remove EFI boot entry:

```sh
sudo efibootmgr
sudo efibootmgr -b XXXX -B
```