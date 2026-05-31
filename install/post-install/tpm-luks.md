# Auto-Unlock LUKS using TPM
https://wiki.archlinux.org/title/Systemd-cryptenroll

Unlock LUKS2 encrypted `/` using TPM. The TPM releases the unlock key only when the trusted boot chain is intact. If validation fails, the system falls back to the normal LUKS password prompt.

## Set Disk Variables

Set disk variables for either a SATA or NVMe disk.

SATA

Example disk: `sda`

```sh
export DISK="/dev/sda"
export ESP_PART="1"
export ROOT_PART="2"

export ESP_DISK="${DISK}${ESP_PART}"
export ROOT_DISK="${DISK}${ROOT_PART}"
```

## Install Required Packages

Install TPM utilities:

```sh
sudo pacman -S tpm2-tools
```

## Verify TPM2 Support

Check TPM availability:

```sh
systemd-cryptenroll --tpm2-device=list
```

Verify TPM support:

```sh
sudo systemd-analyze has-tpm2
```

Expected output:

```text
yes
```

## List keyslots
systemd-cryptenroll can list the keyslots in a LUKS device, similar to cryptsetup luksDump, but in a more user-friendly format.
```sh
sudo systemd-cryptenroll $ROOT_DISK
```

## Erasing keyslots
```sh
sudo systemd-cryptenroll $ROOT_DISK --wipe-slot=SLOT
```

## Enroll TPM2 Unlock
Enroll TPM-bound unlock token:

```sh 
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 $ROOT_DISK
```
[PCR States](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers)

## Configure mkinitcpio

Edit:
```text
/etc/mkinitcpio.conf
```

```bash
sudo sed -i \
-e 's/^MODULES=.*/MODULES=(tpm_tis tpm_crb)/' \
-e 's|^BINARIES=.*|BINARIES=(/usr/lib/systemd/systemd-tpm2-setup)|' \
-e 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt resume filesystems fsck)/' \
/etc/mkinitcpio.conf
```

Edit:

```text
/etc/kernel/cmdline
```

```sh
UUID=$(sudo blkid -s UUID -o value "$ROOT_DISK")
OFFSET=$(sudo btrfs inspect-internal map-swapfile -r /swap/swapfile)

echo "rd.luks.name=${UUID}=root rd.luks.options=${UUID}=tpm2-device=auto root=/dev/mapper/root rootflags=subvol=@ rw noatime resume=/dev/mapper/root resume_offset=${OFFSET}" | sudo tee /etc/kernel/cmdline >/dev/null
```

## Configure crypttab.initramfs

For redundancy, also create:

```text
/etc/crypttab.initramfs
```

Contents:

```sh
echo "root UUID=$(sudo blkid -s UUID -o value $ROOT_DISK) none tpm2-device=auto" | sudo tee /etc/crypttab.initramfs
```


## Rebuild Initramfs

Rebuild initramfs:

```sh
sudo mkinitcpio -P
```

## Re-Sign EFI Binaries

After rebuilding UKIs:

```sh
sudo /usr/local/bin/sign-efi
```

## Verify Current Kernel Command Line

After boot:

```sh
cat /proc/cmdline
```

Verify:

```text
rd.luks.name=
rd.luks.options=
```
are present.

## Verify TPM Enrollment

List enrolled TPM tokens:

```sh
sudo systemd-cryptenroll $ROOT_DISK
cryptsetup luksDump $ROOT_DISK
systemd-cryptsetup attach mapping_name $ROOT_DISK none tpm2-device=auto
```