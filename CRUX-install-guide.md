# CRUX 3.8 Installation Guide
### For ASRock B870 Taichi Lite / Ryzen 9800X3D / RX 7800 XT / Dual NVMe

A simplified, step-by-step guide tailored to this specific hardware.
Focuses on getting CRUX installed and **booting reliably via UEFI + GRUB**.

---

## Table of Contents

1. [Before You Start](#1-before-you-start)
2. [BIOS/UEFI Settings](#2-biosuefi-settings)
3. [Boot the CRUX ISO](#3-boot-the-crux-iso)
4. [Identify Your Target NVMe Drive](#4-identify-your-target-nvme-drive)
5. [Partition the NVMe Drive](#5-partition-the-nvme-drive)
6. [Format with Labels and Mount](#6-format-with-labels-and-mount)
7. [Run Setup — Package Selection](#7-run-setup--package-selection)
8. [Chroot Into Your New System](#8-chroot-into-your-new-system)
9. [Configure fstab (Using Labels)](#9-configure-fstab-using-labels)
10. [Build the Kernel](#10-build-the-kernel)
11. [Install GRUB (UEFI)](#11-install-grub-uefi)
12. [Basic System Configuration](#12-basic-system-configuration)
13. [Exit Chroot and Reboot](#13-exit-chroot-and-reboot)
14. [Troubleshooting Boot Issues](#14-troubleshooting-boot-issues)

---

## IMPORTANT: The Dual NVMe Drive-Swapping Problem

**You have two 1TB NVMe drives. Each boot, the kernel may assign them in
a different order** — the drive that was `nvme0n1` last boot might be
`nvme1n1` this boot. This is a **known issue** caused by PCIe enumeration
order, which is not guaranteed to be consistent.

**This means you CANNOT safely hardcode `/dev/nvme0n1p2` in your fstab
or grub config.** One day it will point at the wrong drive.

**The fix: use filesystem LABELS (or UUIDs) everywhere.** Labels are
short names you assign when formatting (e.g., `CRUX_ROOT`). Unlike device
paths, labels are stored on the partition itself and never change regardless
of which NVMe slot the kernel discovers first.

This guide uses **labels** throughout. They're easier to read and type than
UUIDs. The labels we'll use:

| Label | Partition | Purpose |
|---|---|---|
| `CRUX_EFI` | ESP (512 MiB, FAT32) | EFI bootloader files |
| `CRUX_ROOT` | Root (ext4) | System root `/` |
| `CRUX_SWAP` | Swap (4 GiB) | Swap space |
| `CRUX_HOME` | Home (ext4) | User data `/home` |

---

## 1. Before You Start

**Download and verify the ISO:**

```bash
shasum -a 256 crux-3.8.iso
# Compare output with crux-3.8.sha256 from the download site
```

**Write to USB flash drive** (from an existing Linux system):

```bash
sudo dd if=crux-3.8.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

> Replace `/dev/sdX` with your USB drive — use `lsblk` to identify it. Be careful.

---

## 2. BIOS/UEFI Settings

Enter BIOS by pressing **F2** (or **Del**) during POST.

Verify/change these settings:

| Setting | Value | Where to find it |
|---|---|---|
| **Secure Boot** | Disabled | Security → Secure Boot |
| **CSM (Compatibility Support Module)** | Disabled | Boot → CSM |
| **Fast Boot** | Disabled | Boot → Fast Boot |
| **Boot Mode** | UEFI Only | Boot |
| **Above 4G Decoding** | Enabled | Advanced → AMD CBS or similar |
| **Re-Size BAR** | Enabled | Advanced (helps GPU performance) |

> **Critical:** CSM must be **Disabled**. If CSM is enabled, the board may attempt
> legacy MBR booting and skip the EFI System Partition entirely. This is a common
> reason GRUB doesn't appear after install.

Set your USB flash drive as the first boot device, save, and reboot.

---

## 3. Boot the CRUX ISO

Press **Enter** at the boot prompt. You'll be logged in as root on tty1.

Verify your NVMe drives are detected:

```bash
lsblk
```

You should see **two** NVMe drives, something like:

```
nvme0n1    259:0    0  953.9G  0 disk
nvme1n1    259:1    0  953.9G  0 disk
```

> **NVMe naming:** Drives are `/dev/nvme0n1` and `/dev/nvme1n1`, NOT `/dev/sda`.
> Partitions use a `p` suffix: `/dev/nvme0n1p1`, `/dev/nvme0n1p2`, etc.

---

## 4. Identify Your Target NVMe Drive

Since your two NVMe drives can swap names between boots, you need to figure
out which physical drive you want to install CRUX on **right now**.

```bash
# Show both drives with model numbers and serial numbers:
lsblk -o NAME,SIZE,MODEL,SERIAL

# Or get detailed info:
nvme list
```

**Write down the model/serial of the drive you choose.** If you ever need to
recover and the drives have swapped, you can use `lsblk -o NAME,MODEL,SERIAL`
to find the right one again.

For the rest of this guide, we'll call whichever drive you chose `TARGETDISK`.
Set a variable so you don't have to keep typing it:

```bash
# CHANGE THIS to match YOUR target drive right now:
export DISK=/dev/nvme0n1
# (or /dev/nvme1n1 — whichever you identified above)
```

> Verify: `lsblk $DISK` should show your chosen drive.

---

## 5. Partition the NVMe Drive

We'll create 4 partitions:

| # | Size | Type | Purpose |
|---|---|---|---|
| 1 | 512 MiB | EFI System | ESP (bootloader) |
| 2 | 100 GiB | Linux filesystem | Root `/` |
| 3 | 4 GiB | Linux swap | Swap |
| 4 | Remaining | Linux filesystem | `/home` |

> **Layout suggestion:** If your NVMe is 1TB, something like 512M EFI + 100G root
> + 4G swap + ~860G home works well. Adjust to your preference.

```bash
cfdisk $DISK
```

Inside cfdisk (TUI — use arrow keys to navigate, Enter to select):

```
1. If prompted for a label type, select "gpt"
   (This creates a new GPT partition table — WIPES ALL DATA ON THIS DRIVE)

2. Select "New" → enter 512M → then select "Type" → choose "EFI System"
   This creates partition 1 (ESP)

3. Arrow-down to the free space → select "New" → enter 100G
   This creates partition 2 (root, defaults to "Linux filesystem")

4. Arrow-down to the free space → select "New" → enter 4G
   Then select "Type" → choose "Linux swap"
   This creates partition 3 (swap)

5. Arrow-down to the free space → select "New" → press Enter to accept remaining space
   This creates partition 4 (home, defaults to "Linux filesystem")

6. Select "Write" → type "yes" to confirm → select "Quit"
```

---

## 6. Format with Labels and Mount

**This is where we assign labels.** Labels are the key to surviving the NVMe
name-swapping between boots.

```bash
# Format ESP as FAT32 with label "CRUX_EFI"
mkfs.fat -F 32 -n CRUX_EFI ${DISK}p1

# Format root as ext4 with label "CRUX_ROOT"
mkfs.ext4 -L CRUX_ROOT ${DISK}p2

# Format swap with label "CRUX_SWAP"
mkswap -L CRUX_SWAP ${DISK}p3

# Format home as ext4 with label "CRUX_HOME"
mkfs.ext4 -L CRUX_HOME ${DISK}p4
```

**Verify your labels were set:**

```bash
blkid ${DISK}p1 ${DISK}p2 ${DISK}p3 ${DISK}p4
```

You should see `LABEL="CRUX_ROOT"`, `LABEL="CRUX_EFI"`, etc. in the output.
Also note the `UUID=` values — we won't need them since we're using labels,
but they're a good backup reference.

**Now mount everything:**

```bash
# Activate swap
swapon ${DISK}p3

# Mount root
mount ${DISK}p2 /mnt

# Mount ESP
mkdir -p /mnt/boot/efi
mount ${DISK}p1 /mnt/boot/efi

# Mount home
mkdir -p /mnt/home
mount ${DISK}p4 /mnt/home
```

---

## 7. Run Setup — Package Selection

```bash
setup
```

The setup script will ask where your root partition is mounted (`/mnt`).

**Packages to select:**

### From `core` (select ALL):
Everything in core is recommended.

### From `opt` — essential packages for this hardware:

| Package | Why it's needed |
|---|---|
| **`grub2-efi`** | GRUB bootloader for UEFI systems |
| **`efibootmgr`** | Creates UEFI boot entries in NVRAM |
| **`linux-firmware`** | **ESSENTIAL** — GPU firmware for RX 7800 XT. Without this, you get NO display output after booting. Also has AMD CPU microcode. |
| **`dosfstools`** | Needed to create/check FAT32 filesystems (ESP) |
| **`wpa_supplicant`** | Needed for WiFi (if you ever want wireless) |
| **`wireless-tools`** | WiFi utilities |

> **Do NOT select `grub2`** (the non-EFI version) — that's for legacy BIOS only.
> Select **`grub2-efi`** specifically.

Wait for installation to complete. The last line should say `"0 error(s)"`.

---

## 8. Chroot Into Your New System

The fastest way:

```bash
setup-chroot
```

Or manually (if `setup-chroot` doesn't work):

```bash
mount --bind /dev /mnt/dev
mount --bind /tmp /mnt/tmp
mount --bind /run /mnt/run
mount -t proc proc /mnt/proc
mount -t sysfs none /mnt/sys
mount -t devpts -o noexec,nosuid,gid=tty,mode=0620 devpts /mnt/dev/pts

# THIS LINE IS CRITICAL FOR UEFI — without it efibootmgr won't work:
mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars

chroot /mnt /bin/bash
```

**Set the root password:**

```bash
passwd
```

---

## 9. Configure fstab (Using Labels)

Edit `/etc/fstab`:

```bash
vim /etc/fstab    # or: nano /etc/fstab
```

Use **labels** instead of device paths. This is critical because your NVMe
drives swap names between boots:

```
# <device>                <mountpoint>   <type>   <options>                       <dump> <pass>
LABEL=CRUX_ROOT           /              ext4     defaults                        0      1
LABEL=CRUX_EFI            /boot/efi      vfat     defaults                        0      0
LABEL=CRUX_SWAP           none           swap     sw                              0      0
LABEL=CRUX_HOME           /home          ext4     defaults                        0      2
devpts                    /dev/pts       devpts   noexec,nosuid,gid=tty,mode=0620 0      0
shm                       /dev/shm       tmpfs    defaults                        0      0
```

> **Why labels?** If next boot your CRUX drive becomes `nvme1n1` instead of
> `nvme0n1`, a line like `/dev/nvme0n1p2 /` would mount the WRONG drive (or
> fail entirely). `LABEL=CRUX_ROOT` always finds the right partition, no matter
> which NVMe slot the kernel enumerated first.

---

## 10. Build the Kernel

This is the most important step for getting a bootable system.

```bash
cd /usr/src/linux-6.12.23
make menuconfig
```

> **Tip:** The CRUX setup installs a default `.config` that has many options
> already enabled. Start from that. You mainly need to **verify** the items
> below are enabled. In menuconfig, press `/` to search for a config name
> (e.g., `BLK_DEV_NVME`).

### MUST-HAVE kernel options for your hardware

#### NVMe support (REQUIRED — root is on NVMe):

```
Device Drivers → NVME Support →
    <*> NVM Express block device                [CONFIG_BLK_DEV_NVME=y]
```

> **This is the #1 thing missing from a default SATA-oriented config.**
> The handbook examples show `CONFIG_SATA_AHCI=y` and `CONFIG_BLK_DEV_SD=y` —
> those are for SATA drives. Your NVMe needs `CONFIG_BLK_DEV_NVME=y`.
> It **MUST be `=y` (built-in)**, not `=m` (module), unless you use an initramfs.

#### Filesystem (REQUIRED — must match what you formatted):

```
File systems →
    <*> The Extended 4 (ext4) filesystem        [CONFIG_EXT4_FS=y]
```

#### VFAT/FAT support (REQUIRED — for mounting the ESP):

```
File systems → DOS/FAT/EXFAT/NT Filesystems →
    <*> VFAT (Windows-95) fs support            [CONFIG_VFAT_FS=y]

File systems → Native language support →
    <*> Codepage 437 (United States, Canada)    [CONFIG_NLS_CODEPAGE_437=y]
    <*> NLS ISO 8859-1 (Latin 1)                [CONFIG_NLS_ISO8859_1=y]
```

> Without these, the kernel can't mount the FAT32 EFI partition and you may
> see mount errors on boot.

#### EFI support (REQUIRED for UEFI boot):

```
Processor type and features →
    [*] EFI runtime service support             [CONFIG_EFI=y]
    [*] EFI stub support                        [CONFIG_EFI_STUB=y]

Firmware Drivers → EFI (Extensible Firmware Interface) Support →
    <*> EFI Variable Support via sysfs          [CONFIG_EFI_VARS=y]
```

#### Partition table (REQUIRED for GPT):

```
Enable the block layer →
    Partition Types →
        [*] Advanced partition selection
        [*] EFI GUID Partition support          [CONFIG_EFI_PARTITION=y]
```

#### AMD CPU:

```
Processor type and features →
    [*] AMD ACPI2Platform devices support       [CONFIG_X86_AMD_PLATFORM_DEVICE=y]
    [*] CPU microcode loading support           [CONFIG_MICROCODE=y]
    [*]   AMD microcode loading support         [CONFIG_MICROCODE_AMD=y]

Power management →
    CPU Frequency scaling →
        <*> AMD Processor P-State driver        [CONFIG_X86_AMD_PSTATE=y]
```

#### AMD GPU (RX 7800 XT) — module is fine, loads from disk after boot:

```
Device Drivers → Graphics support →
    <M> AMD GPU                                 [CONFIG_DRM_AMDGPU=m]
    [*]   AMD DC - Enable new display engine    [CONFIG_DRM_AMD_DC=y]
    [*]     DC support for floating point       [CONFIG_DRM_AMD_DC_FP=y]

Device Drivers → Generic Driver Options → Firmware Loader →
    [*] Enable the firmware sysfs fallback mechanism  [CONFIG_FW_LOADER=y]
```

> The GPU driver can be a module (`=m`) — it's not needed to boot, just for
> display. You need `linux-firmware` installed for the GPU to work.

#### Wired Ethernet (for your onboard NIC):

The B870 Taichi Lite has an Intel I226-V or Realtek 2.5GbE NIC. Enable:

```
Device Drivers → Network device support → Ethernet driver support →

    # For Intel I226-V (most likely on Taichi):
    <*> Intel(R) Ethernet Connection I225/I226 (igc)   [CONFIG_IGC=y]

    # If Realtek 2.5GbE instead:
    <*> Realtek 8169/8168/8101/8125 (r8169)            [CONFIG_R8169=y]
```

> Enable both if you're unsure — the unused one won't hurt.

#### WiFi (if you have a WiFi card or plan to add one):

WiFi depends on the exact card. Most common in desktop builds:

```
Networking support →
    [*] Wireless →
        <M> cfg80211 - wireless configuration API     [CONFIG_CFG80211=m]
        <M> Generic IEEE 802.11 Networking Stack       [CONFIG_MAC80211=m]

# Then enable the driver for your specific card under:
Device Drivers → Network device support → Wireless LAN →
    # Intel WiFi: <M> Intel Wireless WiFi (iwlwifi)
    # MediaTek:   <M> MediaTek MT76x2U/MT7663U/MT7921U
    # etc.
```

> You'll also need `linux-firmware` (already selected) for WiFi firmware blobs.

#### USB support (REQUIRED — for USB ports, flash drives, peripherals):

```
Device Drivers → USB support →
    <*> Support for Host-side USB                      [CONFIG_USB_SUPPORT=y]
    <*> xHCI HCD (USB 3.0) support                    [CONFIG_USB_XHCI_HCD=y]
    <*>   xHCI support for AMD SoCs (Ryzen/EPYC)      [CONFIG_USB_XHCI_PCI=y]
    <*> EHCI HCD (USB 2.0) support                    [CONFIG_USB_EHCI_HCD=y]
    <*> USB Mass Storage support                       [CONFIG_USB_STORAGE=y]
```

> `xHCI` covers USB 3.x ports; `EHCI` covers USB 2.0. The B870 Taichi Lite
> has both. `USB_STORAGE` is needed for USB flash drives and external disks.
> All of these can be built-in (`=y`) — they're small and always useful.

#### DVD/Blu-ray burner (optical drive support):

```
Device Drivers → SCSI device support →
    <*> SCSI CDROM support                             [CONFIG_BLK_DEV_SR=y]

Device Drivers → ATA/ATAPI/MFM/RLL support (or Serial ATA/Parallel ATA) →
    <*> AHCI SATA support                              [CONFIG_SATA_AHCI=y]

File systems → CD-ROM/DVD Filesystems →
    <*> ISO 9660 CDROM file system support             [CONFIG_ISO9660_FS=y]
    [*]   Microsoft Joliet CDROM extensions             [CONFIG_JOLIET=y]
    <*> UDF file system support                        [CONFIG_UDF_FS=y]
```

> `BLK_DEV_SR` is the SCSI CD-ROM driver — it handles DVD and Blu-ray drives
> too (Linux treats optical drives as SCSI devices regardless of interface).
> `SATA_AHCI` you likely already have enabled for your NVMe/SATA controller.
> `ISO9660` and `UDF` let you read data discs and movie/video discs.
>
> **To burn discs**, you'll also need userspace tools after the system is
> running. Install them with `prt-get`:
>
> ```bash
> prt-get depinst cdrtools      # provides cdrecord, mkisofs
> prt-get depinst dvd+rw-tools  # provides growisofs for DVD+R/RW
> ```

### Build the kernel

```bash
cd /usr/src/linux-6.12.23
make all
make modules_install
cp arch/x86/boot/bzImage /boot/vmlinuz-6.12.23
cp System.map /boot/
```

> Building the kernel takes time — on a 9800X3D it should be reasonably fast,
> but expect 10-30 minutes depending on how many options are enabled.

---

## 11. Install GRUB (UEFI)

**This is where previous installs likely went wrong.** Follow each step exactly.

### 11.1. Make sure the ESP is mounted inside the chroot:

```bash
# Check if already mounted:
mount | grep efi

# If not mounted:
mount LABEL=CRUX_EFI /boot/efi
```

### 11.2. Install GRUB — run BOTH commands:

```bash
# Standard install — creates NVRAM boot entry via efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=CRUX

# Fallback install — copies GRUB to the universal path ALL UEFI firmwares search
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable
```

> **Why two commands?** ASRock motherboards are known to sometimes **ignore or
> lose custom NVRAM boot entries**. The first command creates a proper entry at
> `EFI/CRUX/grubx64.efi`. The second copies GRUB to `EFI/BOOT/BOOTX64.EFI` —
> the **universal fallback path** that every UEFI firmware is required to check.
> This is very likely why your previous installs didn't show up in the boot menu.

### 11.3. Generate the GRUB config:

```bash
grub-mkconfig > /boot/grub/grub.cfg
```

### 11.4. VERIFY the generated config:

```bash
grep -E "root=|LABEL=|UUID=" /boot/grub/grub.cfg
```

Check that it points to your CRUX root partition. `grub-mkconfig` often
auto-detects the drive, but with two NVMe drives it might pick the wrong one
or use a `/dev/nvme*` path that will break when drives swap.

### 11.5. Write a manual grub.cfg (RECOMMENDED for dual-NVMe setups):

Because `grub-mkconfig` may generate device paths that break when your NVMe
drives swap, a **hand-written config using labels** is more reliable:

```bash
cat > /boot/grub/grub.cfg << 'EOF'
set timeout=10
set default=0

# Search for the partition labeled CRUX_ROOT, regardless of device name
search --label CRUX_ROOT --set=root --no-floppy

menuentry "CRUX 3.8" {
    linux /boot/vmlinuz-6.12.23 root=LABEL=CRUX_ROOT rw quiet
}

menuentry "CRUX 3.8 (single-user)" {
    linux /boot/vmlinuz-6.12.23 root=LABEL=CRUX_ROOT rw single
}

menuentry "CRUX 3.8 (verbose boot)" {
    linux /boot/vmlinuz-6.12.23 root=LABEL=CRUX_ROOT rw
}
EOF
```

> **Why `root=LABEL=CRUX_ROOT` ?** This tells the kernel to find the root
> filesystem by its label instead of a device path. Even if the NVMe drives
> swap to different names, the label stays on the partition. Same idea as the
> fstab we wrote earlier.
>
> The `search --label` line tells GRUB where to find the kernel file itself.

### 11.6. Verify the EFI files are in place:

```bash
ls -la /boot/efi/EFI/CRUX/
# Should show: grubx64.efi

ls -la /boot/efi/EFI/BOOT/
# Should show: BOOTX64.EFI

efibootmgr -v
# Should show a "CRUX" entry in the boot list
```

> If `efibootmgr -v` gives "EFI variables not supported", you forgot to
> mount efivars in step 8. Run:
> `mount --bind /sys/firmware/efi/efivars /sys/firmware/efi/efivars`
> Then re-run both `grub-install` commands.

---

## 12. Basic System Configuration

### 12.1. Timezone, hostname, services — edit `/etc/rc.conf`:

```bash
vim /etc/rc.conf
```

Example settings:

```bash
FONT=default
KEYMAP=us
TIMEZONE=America/New_York          # Change to yours: ls /usr/share/zoneinfo/
HOSTNAME=crux
SYSLOG=sysklogd
SERVICES=(crond lo net)
```

### 12.2. Wired Network (Spectrum DHCP)

Spectrum uses DHCP, so the default `/etc/rc.d/net` config should work
out of the box — it's already set to `TYPE="DHCP"`.

The only thing you may need to change is the network device name. Find it:

```bash
ip link
```

It will likely be something like `enp14s0` or `enp6s0`. If the default
`net` script works with `dhcpcd` (which uses all interfaces by default),
you may not need to change anything. But if you want to be explicit, edit
`/etc/rc.d/net` and set:

```bash
DEV=enp14s0    # ← replace with your actual device name from "ip link"
```

Make sure `net` is in the SERVICES array in `/etc/rc.conf` (it is in the
example above).

### 12.3. WiFi Setup (for future use)

You need `wpa_supplicant` installed (selected during setup). To connect:

```bash
# Find your wireless interface name:
ip link
# It'll be something like wlan0 or wlp5s0

# Create a config file:
wpa_passphrase "YourNetworkName" "YourPassword" > /etc/wpa_supplicant-wlan0.conf

# Test the connection (replace wlan0 with your interface):
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant-wlan0.conf &
dhcpcd wlan0
```

To make WiFi start at boot, add `wlan` to the SERVICES array in `/etc/rc.conf`:

```bash
SERVICES=(crond lo wlan)
```

The `wlan` service script (from `wpa_supplicant` package) manages both
`wpa_supplicant` and DHCP on your wireless interface.

> To switch between wired and WiFi, use `net` for wired and `wlan` for wireless
> in the SERVICES array. You can have both: `SERVICES=(crond lo net wlan)`.

### 12.4. DNS

With Spectrum DHCP, DNS is provided automatically by `dhcpcd`. If you
have issues, you can manually set DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

### 12.5. Generate locale:

```bash
vim /etc/locale.gen
# Uncomment the line: en_US.UTF-8 UTF-8

/usr/sbin/locale-gen
```

### 12.6. Create a regular user:

```bash
useradd -m -G wheel,audio,video -s /bin/bash yourusername
passwd yourusername
```

---

## 13. Exit Chroot and Reboot

```bash
# Exit the chroot
exit

# Unmount everything (the -R flag does it recursively)
umount -R /mnt

# Turn off swap
swapoff -a

# Reboot
reboot
```

Remove the USB drive when the system restarts.

**At the BIOS boot screen**, you should see either:
- "CRUX" as a boot option, or
- The system boots directly into GRUB (via the fallback `BOOTX64.EFI`)

If GRUB appears with the menu, select "CRUX 3.8" and press Enter.

---

## 14. Troubleshooting Boot Issues

### CRUX doesn't appear in the boot menu / only UEFI settings show

**Most likely cause:** GRUB was not installed to the fallback path.

**Fix:** Boot from the CRUX USB, then:

```bash
# Find your CRUX drive (look at model/serial, don't trust the name):
lsblk -o NAME,SIZE,MODEL,SERIAL,LABEL

# Mount using labels (these always work regardless of nvme name):
mount LABEL=CRUX_ROOT /mnt
mount LABEL=CRUX_EFI /mnt/boot/efi

# Chroot
setup-chroot    # or do the manual mounts from step 8

# Inside chroot — install GRUB to the fallback path:
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable

# Verify:
ls /boot/efi/EFI/BOOT/BOOTX64.EFI
```

### GRUB appears but says "error: no such device" or can't find kernel

The `grub.cfg` has a wrong root device path. This is often caused by the
NVMe drive-swapping problem.

**Fix:** Use the manual label-based `grub.cfg` from step 11.5. Boot from USB,
chroot, and write the manual config.

### Kernel panics: "VFS: Unable to mount root fs"

Two possible causes:

**Cause A — NVMe driver not built in:**

```bash
# In chroot:
grep CONFIG_BLK_DEV_NVME /usr/src/linux-6.12.23/.config
# Must show: CONFIG_BLK_DEV_NVME=y  (NOT =m, NOT "is not set")

# If wrong, reconfigure and rebuild:
cd /usr/src/linux-6.12.23
make menuconfig     # Enable NVMe as built-in <*>
make all && make modules_install
cp arch/x86/boot/bzImage /boot/vmlinuz-6.12.23
```

**Cause B — NVMe drives swapped and grub.cfg uses device path:**

The kernel was told `root=/dev/nvme0n1p2` but the drive is now `nvme1n1`.
Fix by switching to `root=LABEL=CRUX_ROOT` in grub.cfg (step 11.5).

### Screen goes black / no display after booting

**Cause:** Missing GPU firmware (`linux-firmware` package not installed).

```bash
# In chroot:
pkginfo -i | grep linux-firmware

# If not installed, mount the USB and install it:
mount /dev/sdX /media    # your USB drive
pkgadd /media/crux/opt/linux-firmware#*.pkg.tar.*
```

### Wrong partition mounts after NVMe swap

If things seem weird (wrong data in `/home`, etc.), your NVMe drives
swapped and your fstab is using device paths.

**Fix:** Switch fstab to labels (step 9). Boot from USB, chroot, and
update `/etc/fstab` to use `LABEL=` instead of `/dev/nvme*`.

### BIOS "resets" and removes the CRUX boot entry

ASRock boards sometimes clear custom NVRAM entries. The `--removable` flag
from step 11 is the permanent fix. Additionally:

1. In BIOS → Boot, look for "Add New Boot Option" or "UEFI Hard Disk BBS Priorities"
2. Manually add a boot entry pointing to `EFI/BOOT/BOOTX64.EFI` on your NVMe
3. Set it as first priority

### No internet after booting

```bash
# Check if the interface exists:
ip link

# If the interface shows but is down:
ip link set enp14s0 up     # replace with your device name
dhcpcd enp14s0

# If no ethernet interface appears at all, the kernel driver is missing.
# Chroot and rebuild the kernel with CONFIG_IGC=y or CONFIG_R8169=y (step 10).
```

### How to re-enter chroot from USB (general recovery):

```bash
# Use labels — they work even when NVMe names have swapped:
mount LABEL=CRUX_ROOT /mnt
mount LABEL=CRUX_EFI /mnt/boot/efi
mount LABEL=CRUX_HOME /mnt/home

mount --bind /dev /mnt/dev
mount --bind /tmp /mnt/tmp
mount --bind /run /mnt/run
mount -t proc proc /mnt/proc
mount -t sysfs none /mnt/sys
mount -t devpts -o noexec,nosuid,gid=tty,mode=0620 devpts /mnt/dev/pts
mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars
chroot /mnt /bin/bash
```

> **Notice:** These recovery mount commands use `LABEL=` instead of
> `/dev/nvme*`. This way recovery works no matter which order the
> drives were enumerated.

---

## Partition Layout Summary

| Label | Size | Type | Mount | Purpose |
|---|---|---|---|---|
| `CRUX_EFI` | 512 MiB | FAT32 | `/boot/efi` | EFI bootloader |
| `CRUX_ROOT` | ~100 GiB | ext4 | `/` | System root |
| `CRUX_SWAP` | 4 GiB | swap | (none) | Swap space |
| `CRUX_HOME` | remaining | ext4 | `/home` | User data |

---

## Why It Probably Didn't Boot Before — Summary

In order of likelihood:

1. **GRUB not at the UEFI fallback path** — ASRock boards often only check
   `EFI/BOOT/BOOTX64.EFI`. The `--removable` flag fixes this.

2. **NVMe drives swapped names** — `grub.cfg` or `fstab` had a hardcoded
   `/dev/nvme0n1p2` that pointed at the wrong drive on next boot. Using
   `LABEL=` fixes this permanently.

3. **CSM was enabled in BIOS** — causes legacy booting, skips the ESP.

4. **`CONFIG_BLK_DEV_NVME` not built into kernel** — the handbook defaults
   target SATA drives. NVMe needs its own driver compiled in.

5. **efivars not mounted in chroot** — `efibootmgr` couldn't write the
   NVRAM boot entry.

6. **`grub.cfg` referenced `/dev/sda2`** — `grub-mkconfig` auto-detected
   the USB stick or wrong disk.

---

## Quick Reference Card

```
# Find your CRUX drive when NVMe names have swapped:
lsblk -o NAME,SIZE,MODEL,SERIAL,LABEL

# Mount by label (always works):
mount LABEL=CRUX_ROOT /mnt

# Check boot entries:
efibootmgr -v

# Check kernel config:
grep CONFIG_BLK_DEV_NVME /usr/src/linux-*/.config

# Verify GRUB is at fallback path:
ls /boot/efi/EFI/BOOT/BOOTX64.EFI

# Search menuconfig:
# Press / then type the config name (e.g., BLK_DEV_NVME)
```
