# Secure Boot on Arch Linux using Shim + MOK + systemd-boot + UKI

## Boot Flow

```text
UEFI Firmware
   ↓ trusts Microsoft keys
shimx64.efi
   ↓ trusts your MOK key
grubx64.efi
   ↓ signed systemd-boot
Unified Kernel Image (UKI)
   ↓
Linux boots
```

## Set Disk Variables

Example disk: `sda`

```sh
export DISK="/dev/sda"
export ESP_PART="1"
export ROOT_PART="2"

export ESP_DISK="${DISK}${ESP_PART}"
export ROOT_DISK="${DISK}${ROOT_PART}"
```

## Hyper-V Users

Verify Hyper-V:

```sh
systemd-detect-virt
```

Expected:

```text
microsoft
```

Enable Secure Boot in Hyper-V and use:

```text
Microsoft UEFI Certificate Authority
```

Do not use:

```text
Microsoft Windows
```

## Install Packages

```sh
sudo pacman -S sbsigntools mokutil efibootmgr
```

Install shim:

```sh
git clone https://aur.archlinux.org/shim-signed.git
cd shim-signed
makepkg -si
```

Install systemd-boot:

```sh
sudo bootctl install
```

## Verify Current State

```sh
mokutil --sb-state
```

## Generate MOK Keys

```sh
openssl req -newkey rsa:2048 -nodes -keyout MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=Machine Owner Key/" -out MOK.crt
```

```sh
openssl x509 -outform DER -in MOK.crt -out MOK.cer
```

## Store Keys

```sh
sudo mkdir -p /root/secureboot
```

```sh
sudo mv MOK.key MOK.crt MOK.cer /root/secureboot/
```

```sh
sudo chmod 700 /root/secureboot
```

```sh
sudo sh -c 'chmod 600 /root/secureboot/*'
```

## Install Shim

```sh
sudo mkdir -p /efi/EFI/BOOT
```

```sh
sudo cp /usr/share/shim-signed/shimx64.efi /efi/EFI/BOOT/BOOTX64.EFI
```

```sh
sudo cp /usr/share/shim-signed/mmx64.efi /efi/EFI/BOOT/
```

Verify:

```sh
sha256sum /efi/EFI/BOOT/BOOTX64.EFI
```

```sh
sha256sum /usr/share/shim-signed/shimx64.efi
```

## Create Shim Boot Entry

```sh
sudo efibootmgr --unicode --disk "$DISK" --part "$ESP_PART" --create --label "Shim" --loader '\EFI\BOOT\BOOTX64.EFI'
```

Verify:

```sh
efibootmgr -v
```

## Create Signing Script

```sh
sudo nano /usr/local/bin/sign-efi
```

```bash
#!/usr/bin/env bash
set -euo pipefail

KEY="/root/secureboot/MOK.key"
CERT="/root/secureboot/MOK.crt"

sign() {
    local file="$1"

    [[ -f "$file" ]] || return 0

    if sbverify --cert "$CERT" "$file" >/dev/null 2>&1; then
        echo "Already signed: $file"
        return 0
    fi

    echo "Signing: $file"

    sbsign --key "$KEY" --cert "$CERT" --output "${file}.signed" "$file"

    mv "${file}.signed" "$file"
}

sign /efi/EFI/systemd/systemd-bootx64.efi

cp -af /efi/EFI/systemd/systemd-bootx64.efi /efi/EFI/BOOT/grubx64.efi

for uki in /efi/EFI/Linux/*.efi; do
    [[ -f "$uki" ]] && sign "$uki"
done

echo "Done"
```

```sh
sudo chmod +x /usr/local/bin/sign-efi
```

## Initial Signing

```sh
sudo /usr/local/bin/sign-efi
```

## Verify Signatures

```sh
sbverify --list /efi/EFI/systemd/systemd-bootx64.efi
```

```sh
sbverify --list /efi/EFI/BOOT/grubx64.efi
```

```sh
sbverify --list /efi/EFI/Linux/ArchLinux-linux.efi
```

```sh
sbverify --list /efi/EFI/Linux/ArchLinux-recovery.efi
```

## Verify SBAT

```sh
objdump -j .sbat -s /efi/EFI/systemd/systemd-bootx64.efi
```

```sh
objdump -j .sbat -s /efi/EFI/Linux/ArchLinux-linux.efi
```

## Import MOK

```sh
sudo cp /root/secureboot/MOK.cer /efi/
```

```sh
sudo mokutil --import /efi/MOK.cer
```

Set a temporary password.

Verify:

```sh
mokutil --list-new
```

Expected:

```text
CN=Machine Owner Key
```

## Enroll MOK

Enable Secure Boot and reboot.

In MokManager:

```text
Enroll MOK
Continue
Yes
```

Enter the password used during import.

Reboot again.

## Automatic Re-Signing

Create helper:

```sh
sudo nano /usr/local/bin/rebuild-and-sign-efi
```

```bash
#!/usr/bin/env bash
set -euo pipefail

mkinitcpio -P
/usr/local/bin/sign-efi
```

```sh
sudo chmod +x /usr/local/bin/rebuild-and-sign-efi
```

## mkinitcpio Post Hook

Sign UKIs whenever `mkinitcpio` generates them.

Create hook directory:

```sh
sudo mkdir -p /etc/initcpio/post
```

Create hook:

```sh
sudo nano /etc/initcpio/post/90-sign-efi
```

```bash
#!/usr/bin/env bash

/usr/local/bin/sign-efi
```

Make executable:

```sh
sudo chmod +x /etc/initcpio/post/90-sign-efi
```

### Verify

Rebuild UKIs:

```sh
sudo mkinitcpio -P
```

Verify signatures:

```sh
sbverify --list /efi/EFI/Linux/ArchLinux-linux.efi
```

```sh
sbverify --list /efi/EFI/Linux/ArchLinux-recovery.efi
```

## Kernel Hook

```sh
sudo mkdir -p /etc/pacman.d/hooks
```

```sh
sudo nano /etc/pacman.d/hooks/95-secureboot-rebuild.hook
```

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Path
Target = usr/lib/modules/*
Target = usr/lib/initcpio/*

[Action]
Description = Rebuild UKIs and sign EFI binaries
When = PostTransaction
Exec = /usr/local/bin/rebuild-and-sign-efi
```

## systemd Hook

```sh
sudo nano /etc/pacman.d/hooks/96-systemd-boot-sign.hook
```

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = systemd

[Action]
Description = Re-sign EFI binaries
When = PostTransaction
Exec = /usr/local/bin/sign-efi
```

## Remove Setup

Remove shim:

```sh
sudo pacman -R shim-signed
```

Remove files:

```sh
sudo rm -f /efi/EFI/BOOT/BOOTX64.EFI
```

```sh
sudo rm -f /efi/EFI/BOOT/mmx64.efi
```

```sh
sudo rm -f /efi/EFI/BOOT/grubx64.efi
```

List entries:

```sh
sudo efibootmgr
```

Remove Shim entry:

```sh
sudo efibootmgr -b XXXX -B
```

Replace `XXXX` with the Shim boot number.
