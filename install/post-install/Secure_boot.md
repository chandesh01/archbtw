# Secure Boot on Arch Linux with `sbctl`

https://github.com/Foxboron/sbctl/

## Install `sbctl`

Install the Secure Boot management utility:

```sh
sudo pacman -S sbctl
```

## Verify UEFI Boot Mode

Ensure the system is booted in UEFI mode:

```sh
cat /sys/firmware/efi/fw_platform_size
```

If this file exists and outputs `64` or `32`, the system is using UEFI.

You can also check boot information:

```sh
bootctl status
```

## Enter UEFI Firmware Setup

Reboot your system and enter the firmware setup utility.

Common keys:

- `F2`
- `DEL`
- `ESC`
- `F10`

Locate the **Secure Boot** section.

## Clear Existing Secure Boot Keys

To use your own keys, the firmware must enter **Setup Mode**.

Inside the UEFI settings:

- Disable Secure Boot temporarily (if required)
- Choose:
  - **Clear Secure Boot Keys**
  - **Delete All Keys**
  - **Reset to Setup Mode**

This removes:

- Platform Key (PK)
- Key Exchange Keys (KEK)
- Signature Database (db/dbx)

Save changes and boot back into Linux.

## Verify Setup Mode

Check Secure Boot status:

```sh
sbctl status
```

Expected output:

```text
Setup Mode: ✓ Enabled
Secure Boot: ✗ Disabled
```

If `Setup Mode` is enabled, continue.

## Generate Secure Boot Keys

Create your Secure Boot signing keys:

```sh
sudo sbctl create-keys
```

The keys are stored under:

```text
/var/lib/sbctl/keys/
```

## Enroll Keys into Firmware

Enroll the generated keys into UEFI:

```sh
sudo sbctl enroll-keys --microsoft --firmware-builtin
```
The `--microsoft` flag also installs Microsoft's Secure Boot certificates.

This allows:

- Dual booting Windows
- Booting Microsoft-signed EFI binaries
- Better compatibility with third-party tools

Without this flag, only binaries signed with your own keys will boot.

## Verify Unsigned EFI Files

Check which EFI binaries are unsigned:

```sh
sbctl verify
```
Signed each the efi

## Sign Unified Kernel Images (UKI)

Sign your Unified Kernel Images:

```sh
sudo sbctl sign -s /efi/EFI/Linux/arch-linux.efi
sudo sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
```

The `-s` flag registers the files for automatic re-signing after updates.

## Sign `vmlinuz-linux` (Optional but Recommended)

If you use a traditional boot setup or want additional compatibility, sign the kernel image directly:

```sh
sudo sbctl sign -s /boot/vmlinuz-linux
```

For additional kernels:

```sh
sudo sbctl sign -s /boot/vmlinuz-linux-lts
sudo sbctl sign -s /boot/vmlinuz-linux-zen
```

## Sign Bootloader EFI Files

### systemd-boot

```sh
sudo sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
```

### GRUB

```sh
sudo sbctl sign -s /efi/EFI/GRUB/grubx64.efi
```

### Fallback Bootloader

```sh
sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
```

## Automatically Re-sign Files After Updates

Enable automatic signing support:

```sh
sudo systemctl enable --now sbctl.service
```

Or manually sign all registered files:

```sh
sudo sbctl sign-all
```

## Verify Signatures

Verify all tracked files are signed:

```sh
sbctl verify
```

Expected output:

```text
✓ /efi/EFI/Linux/arch-linux.efi is signed
✓ /boot/vmlinuz-linux is signed
```

List all tracked files:

```sh
sudo sbctl list-files
```

## Enable Secure Boot

Reboot into the UEFI firmware again.

Inside Secure Boot settings:

- Enable **Secure Boot**

Save and reboot.

## Confirm Secure Boot is Enabled

Back in Linux:

```sh
sbctl status
bootctl status
mokutil --sb-state
```