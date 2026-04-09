---
title: "Build and Run on kernel using QEMU"
description:  "Showcase two ways to run custom kernel using QEMU..."
date: 2026-04-08T16:10:31-07:00
image: 
categories:
    - Linux
tags:
    - Kernel
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

Just for personal note, haven't test on others.

## Kernel Config
 
```bash
make defconfig CC=clang
```

if want to include other module which is not enable be default in defconfig. run for e.g.
```basg
./scripts/config --module CONFIG_MAC80211_HWSIM
./scripts/config --module CONFIG_IWLWIFI=y
./scripts/config --module CONFIG_IWLWIFI_LEDS=y
./scripts/config --module CONFIG_IWLDVM=m
./scripts/config --module CONFIG_IWLMVM=m
```

## build-kernel.sh
 
```bash
#!/bin/bash
set -e
 
make CC=clang -j$(nproc)
python3 scripts/clang-tools/gen_compile_commands.py
 
echo "Done — bzImage ready at arch/x86/boot/bzImage"
```
 
For subsequent small changes just run `make CC=clang -j$(nproc)` directly.
Make only recompiles changed files, so a single `.c` change takes a few seconds.
 
---
 
## Raw Kernel + Busybox Initramfs
 
QEMU loads the kernel and a cpio archive, the kernel unpacks
it into a tmpfs, runs `/init`, and drops you into a busybox shell.
 
### build-initramfs.sh
 
```bash
#!/bin/bash
set -e
 
KBUILD=$(pwd)
 
rm -rf initramfs
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev,lib/modules}
 
cp /usr/bin/busybox initramfs/bin/
cd initramfs/bin
./busybox --install .
cd "$KBUILD"
 
cat > initramfs/init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev 2>/dev/null || mdev -s
 
echo "Kernel $(uname -r) booted!"
exec /bin/sh
EOF
 
chmod +x initramfs/init
 
cd initramfs
find . | cpio -o -H newc | gzip > ../initramfs.cpio.gz
cd ..
 
echo "Done — initramfs.cpio.gz ready"
```
 
### boot.sh
 
```bash
#!/bin/bash
set -e
 
KBUILD=$(pwd)
 
qemu-system-x86_64 \
    -kernel "$KBUILD/arch/x86/boot/bzImage" \
    -initrd "$KBUILD/initramfs.cpio.gz" \
    -append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 rdinit=/init nokaslr" \
    -nographic \
    -serial mon:stdio \
    -device e1000,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -m 512M \
    -no-reboot
```
 
### Rebuild
 
```bash
bash build-kernel.sh
bash build-initramfs.sh
bash boot.sh
```
 
Exit QEMU with `Ctrl-A X` or run `shutdown`.

---
 
## Full Arch Linux Rootfs
 
When you need `pacman`, `ssh`, or any real tooling, install Arch onto a qcow2 image and
boot your custom kernel directly into it.
 
### Create the Disk
 
```bash
qemu-img create -f qcow2 rootfs.qcow2 10G
```
 
### Install Arch from ISO
 
```bash
qemu-system-x86_64 \
    -cdrom archlinux-x86_64.iso \
    -boot d \
    -drive file=rootfs.qcow2,format=qcow2 \
    -device e1000,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -m 2G \
    -enable-kvm \
    -no-reboot \
    -vnc :0
```
 
Connect with `vncviewer localhost:5900`. Inside the installer:
 
```bash
fdisk /dev/sda << 'EOF'
g
n
1
 
 
w
EOF
 
mkfs.ext4 /dev/sda1
mount /dev/sda1 /mnt
 
# base only, no need linux package
pacstrap /mnt base
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
 
passwd root
 
# Network (optional)
cat > /etc/systemd/network/20-wired.network << 'NET'
[Match]
Name=ens3
 
[Network]
DHCP=yes
NET
 
systemctl enable systemd-networkd systemd-resolved
 
exit
umount -R /mnt
poweroff
```
 
### boot.sh
 
```bash
#!/bin/bash
set -e
 
KBUILD=$(pwd)
 
qemu-system-x86_64 \
    -kernel "$KBUILD/arch/x86/boot/bzImage" \
    -drive file="$KBUILD/rootfs.qcow2",format=qcow2 \
    -append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 root=/dev/sda1 rw nokaslr" \
    -nographic \
    -serial mon:stdio \
    -device e1000,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -m 2G \
    -enable-kvm \
    -no-reboot
```
 
SSH in with:

> [!NOTE]
> Need to install and start openssh, and set `PermitRootLogin = yes` in `/etc/ssh/ssh_config`.
 
```bash
ssh -p 2222 root@127.0.0.1
```

### Rebuild
 
```bash
bash build-kernel.sh
bash build-initramfs.sh
bash boot.sh
```
 
Exit QEMU with `Ctrl-A X` or run `shutdown`.
