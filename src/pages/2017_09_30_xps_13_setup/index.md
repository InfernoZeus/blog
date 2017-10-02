---
title: My New XPS 13
date: "2017-09-30"
path: "my-new-xps-13"
---

I've just received my new XPS 13 (9360), which came with Ubuntu 16.04 pre-installed.
Follow along as I convert it to a fully-fledged hacking device (aka running ArchLinux).

### Rough Plan

Before I start, this is the rough order I expect to follow. I wonder how well I can stick to it...

  1. Check out pre-installed bootloader/EFI utils. Do I need to save any stubs, or recovery tools?
  2. Install [rEFInd](http://www.rodsbooks.com/refind/). I love rEFInd. It's so simple, and much better looking than Grub.
  3. Install ArchLinux. I want to try out some type of snap-shotting to allow for rollbacks.
  4. Choose a dotfile manager. This is something I've never used before, but would really love to try some.
  5. Setup Zsh or Fish. Bash is boring - well, not really, but why not try something new!

### EFI Investigation

As this laptop will be used for work, I'd like to keep any recovery options, so let's see what came pre-installed.

#### EFI Boot Options
First off, I started looking at the default EFI boot options, using `efibootmgr`:
```
$ efibootmgr -v
BootCurrent: 0002
Timeout: 0 seconds
BootOrder: 0002
Boot0000* Windows Boot Manager	HD(2,GPT,271ab888-e443-4bc2-bb31-0f6208b29f19,0xe1800,0x32000)/File(\EFI\Microsoft\Boot\bootmgfw.efi)WINDOWS.........x...B.C.D.O.B.J.E.C.T.=.{.9.d.e.a.8.6.2.c.-.5.c.d.d.-.4.e.7.0.-.a.c.c.1.-.f.3.2.b.3.4.4.d.4.7.9.5.}....................
Boot0001* Diskette Drive	BBS(Floppy,Diskette Drive,0x0)..BO
Boot0002* ubuntu	HD(1,GPT,483afb34-eb42-44ce-8a43-28c68c98d52f,0x800,0x12c000)/File(\EFI\ubuntu\shimx64.efi)
Boot0003* USB Storage Device	BBS(USB,USB Storage Device,0x0)..BO
Boot0004* CD/DVD/CD-RW Drive	BBS(CDROM,CD/DVD/CD-RW Drive,0x0)..BO
Boot0005* USB NIC	BBS(128,Realtek PXE B00 D14,0x0)..BO
Boot0006* M.2 PCIe SSD	BBS(HD,P0: KXG50ZNV512G NVMe TOSHIBA 512GB,0x0)..BO
Boot0007* UEFI: KXG50ZNV512G NVMe TOSHIBA 512GB, Partition 1	HD(1,GPT,483afb34-eb42-44ce-8a43-28c68c98d52f,0x800,0x12c000)/File(EFI\boot\bootx64.efi)..BO
```
This tells me that the default boot is `0002` which corresponds to `ubuntu`. I tried booting into each of the other options,
using `efibootmgr -n 000x`, but none of them booted successfully.

#### ESP Contents
Next, I started looking for binaries in the ESP. Searching for *.efi files, or executable files:
```
/boot/efi/EFI/Boot/en-us/bootx64.efi.mui
/boot/efi/EFI/Boot/shimx64.efi
/boot/efi/EFI/dell/bios/recovery/bios_cur.rcv
/boot/efi/EFI/ubuntu/fwupx64.efi
/boot/efi/EFI/ubuntu/grub.cfg
/boot/efi/EFI/ubuntu/grubx64.efi
/boot/efi/EFI/ubuntu/mmx64.efi
/boot/efi/EFI/ubuntu/shimx64.efi
/boot/efi/en-us/bootmgr.efi.mui
```

To test these out, I added a UEFI Shell binary to the efibootmgr:
```
$ sudo wget https://github.com/tianocore/edk2/raw/master/ShellBinPkg/UefiShell/X64/Shell.efi -o /boot/efi/Shell.efi
$ sudo efibootmgr --create --disk /dev/nvme0n1 --part 1 --loader /Shell.efi --label "UEFI Shell"
BootCurrent: 0002
Timeout: 0 seconds
BootOrder: 0008,0002
Boot0000* Windows Boot Manager
Boot0001* Diskette Drive
Boot0002* ubuntu
Boot0003* USB Storage Device
Boot0004* CD/DVD/CD-RW Drive
Boot0005* USB NIC
Boot0006* M.2 PCIe SSD
Boot0007* UEFI: KXG50ZNV512G NVMe TOSHIBA 512GB, Partition 1
Boot0008* UEFI Shell
```
The previous command added UEFI Shell as the default boot, so I reset it, leaving Shell as the next boot:
```
$ sudo efibootmgr -o 0002
$ sudo efibootmgr -o 0008
```
This didn't work either. At this point, I wondered whether Secure Boot was preventing things from working properly.
I disabled Secure Boot in the UEFI options, and the Shell loaded this time. Unfortunately, this didn't lead to much info.

#### Dell Recovery Partition
At this point, I noticed that there's an extra partition on the SSD, roughly 3G in size. This was clearly the Recovery
tools I was looking for, as it mentioned Dell in multiple places. It was being used in the Grub config that came
pre-installed, inserted via /etc/grub.d/99_dell_recovery:
```
#!/bin/bash -e

source /usr/lib/grub/grub-mkconfig_lib

cat << EOF
menuentry "Restore Ubuntu 16.04 to factory state" {
        search --no-floppy --hint '(hd0,gpt2)' --set --fs-uuid C0F9-04A2
        set uuid_options="uuid=C0F9-04A2"
        if [ -s /factory/common.cfg ]; then
            source /factory/common.cfg
        else
            set options="boot=casper automatic-ubiquity noprompt quiet splash nomodeset"
        fi
        if [ -s /factory/post-rts-gfx.cfg ]; then
            source /factory/post-rts-gfx.cfg
        fi
        if [ -s /factory/post-rts-wlan.cfg ]; then
            source /factory/post-rts-wlan.cfg
        fi
        #Support starting from a loopback mount (Only support ubuntu.iso for filename)
        if [ -f /ubuntu.iso ]; then
            loopback loop /ubuntu.iso
            set root=(loop)
            set options="\$options iso-scan/filename=/ubuntu.iso"
        fi
        if [ -n "\${lang}" ]; then
            set options="\$options locale=\$lang"
        fi
        if [ -s /factory/dual_enable ]; then
            set options="\$options dell-recovery/dual_boot=true"
        fi

        linux   /casper/vmlinuz.efi dell-recovery/recovery_type=hdd \$uuid_options \$options
        initrd  /casper/initrd.lz
}
EOF
```
To get this working with rEFInd, I'll have to convert this script, but I'll leave that for later.

### rEFInd Installation

rEFInd installation is very well documented on the tool's homepage, so I won't go into it much here.
I choose to do a manual install as I didn't want to risk losing anything already in the ESP, and
used the binary downloads offered.

I used the default config, and rebooted into rEFInd to see what it could discover automatically.
It found the existing Ubuntu kernel, plus the Grub bootloader. I could boot in both ways, so will
leave custom configuration of rEFInd until I've installed Arch.

### ArchLinux installation

The current partition layout on my disk is as follows:
```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0   477G  0 disk
├─nvme0n1p1 259:1    0   600M  0 part /boot/efi
├─nvme0n1p2 259:2    0     3G  0 part
├─nvme0n1p3 259:3    0 441,6G  0 part /
└─nvme0n1p4 259:4    0  31,8G  0 part [SWAP]
```
The first partition is the ESP, second is the Dell Recovery partition, third is the default Ubuntu install,
and the last is a massive swap partition.

The plan is to leave the first two, add a third small 5GB partition (which I hope to setup as a backup
Arch-based recovery install), add a fourth /boot partition of 1GB, and a final partition as an LVM logical
volume.

#### LVM Links

Useful links for LVM:
- https://forums.suse.com/content.php?133-Understanding-LVM-snapshots-and-how-to-use-them
- https://www.tecmint.com/take-snapshot-of-logical-volume-and-restore-in-lvm/

#### Backup Arch Links

Useful links for a backup Arch install:
- https://bbs.archlinux.org/viewtopic.php?id=155344
- https://bbs.archlinux.org/viewtopic.php?id=140826
- https://bbs.archlinux.org/viewtopic.php?id=208991

