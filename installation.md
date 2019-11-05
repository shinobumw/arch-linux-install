# ArchLinux 重灌記錄
###### tags: `linux`, `note`

> 完成後搬去 github

## Boot

- UEFI with GPT
- Windows 10 installed

## 製作開機USB

- rufus 
- ISO 不行就改用 DD

## 連網路

- 無線就用 `wifi-menu` 連
- `ping 8.8.8.8` 測試有沒有連上

## Disk Partition

`$ fdisk -l` 查看硬碟資訊

`$ fdisk /dev/sda` 分割硬碟，改用`cgdisk`也可以

- `/swap`: sda5 (512M)
- `/root`: sda6 (99.5G)
- `/boot`: 雙系統的話用原本 windows 的 partition，建議大小 260–512 MiB

Formatting
```
# swap
$ mkswap /dev/sda5
# root
$ mkfs.ext4 /dev/sda6
```

Mount
```
$ swapon /dev/sda5
$ mount /dev/sda6 /mnt

# Old Mount point
~~$ mkdir /mnt/boot/efi~~
~~$ mount /dev/sda2 /boot/efi~~

# New Mount point
$ mkdir /mnt/boot
$ mount /dev/sda2 /boot
or
$ mkdir /mnt/efi
$ mount /dev/sda2 /efi
```

## Mirror List

`$ nano /etc/pacman.d/mirrorlist`

把 Taiwan 的 mirror 移到最上面

## 安裝基本系統

`$ pacstrap /mnt  base base-devel`

:::warning
**Note:** The `base` group has been replaced by a metapackage of the same name in order to minimize the installation size.
:::

需要額外安裝的:

- `linux-lts`, `linux-firmware`
- file system (e.g. `e2fsprogs`, `ntfs-3g`)
- text editors (e.g. `nano`, `vim`)
- software necessary for networking (e.g. `networkmanager`)
- packages for accessing documentation in man and info pages: `man-db`, `man-pages` and `texinfo`.

This is what's in the group but not "in the package":
```
$ for pkg in $(comm -23 <(pacman -Sqg base|sort|uniq) <(expac -S %D base | tr " " "\n"|sort|uniq)); do echo $pkg":" $(expac -S %d $pkg); done
cryptsetup: Userspace setup tool for transparent encryption of block devices using dm-crypt
device-mapper: Device mapper userspace library and tools
*dhcpcd: RFC2131 compliant DHCP client daemon
e2fsprogs: Ext2/3/4 filesystem utilities
inetutils: A collection of common network programs
*jfsutils: JFS filesystem utilities
linux: The Linux kernel and modules
linux-firmware: Firmware files for Linux
logrotate: Rotates system logs automatically
lvm2: Logical Volume Manager 2 utilities
man-db: A utility for reading man pages
man-pages: Linux man pages
mdadm: A tool for managing/monitoring Linux md device arrays, also known as Software RAID
nano: Pico editor clone with enhancements
netctl: Profile based systemd network management
perl: A highly capable, feature-rich programming language
*reiserfsprogs: Reiserfs utilities
s-nail: Environment for sending and receiving mail
sysfsutils: System Utilities Based on Sysfs
texinfo: GNU documentation system for on-line information and printed output
usbutils: USB Device Utilities
vi: The original ex/vi text editor
xfsprogs: XFS filesystem utilities
```

##### Reference:

[1] [`base` group replaced by mandatory `base` package - manual intervention required](https://www.archlinux.org/news/base-group-replaced-by-mandatory-base-package-manual-intervention-required/)\
[2] [Arch Linux - News: `base` group replaced by mandatory `base` package - /r/archlinux](https://www.reddit.com/r/archlinux/comments/de1er6/arch_linux_news_base_group_replaced_by_mandatory/)


## 產生 fstab

`$ genfstab -U /mnt >> /mnt/etc/fstab`

## chroot 進入裝好的新系統

`$ arch-chroot /mnt`

## 時區

`$ ln -sf /usr/share/zoneinfo/Asia/Taipei  /etc/localtime`

### 使用 hwclock 來生成 /etc/adjtime

`$ hwclock --systohc`

## 語系

把註解拿掉
```
$ vim /etc/locale.gen

en_US.UTF-8 UTF-8
zh_TW.UTF-8 UTF-8
```
`$ locale-gen`

`$ echo LANG=en_US.UTF-8 > /etc/locale.conf`

## 主機名

```
# 替換myhostname
$ echo myhostname > /etc/hostname 
```
```
$ vim /etc/hosts

127.0.0.1	localhost.localdomain	localhost
::1		localhost.localdomain	localhost
127.0.1.1	myhostname.localdomain	myhostname
```

## 設定 root 密碼

`$ passwd`

## 建立開機映像檔

`$ mkinitcpio -p linux`

`$ pacman -Sy grub os-prober efibootmgr`

os-prober 用來偵測其他系統
```
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```
有可能第一次會偵測不到 Windows，重開機後再用一次就 OK。

## 安裝連網路的工具

`$ pacman -S networkmanager`

不用額外裝的：
- `dhcpcd`, `dialog`
- `wpa_supplicant` (dependency of `networkmanager`)

## 重新開機

```
$ exit
$ umount -R /mnt
$ reboot
```

## 創建用戶

```
# 替換 username
$ useradd -m -G wheel username 
$ passwd username
```
```
$ pacman -S sudo
$ visudo
```
找到`# %wheel ALL=(ALL)ALL`，把#字號拿掉。

`$ reboot`

## GUI

```
$ pacman -S xorg
$ pacman -S xf86-video-intel
# KDE 全家桶，也可以選擇需要的安裝就好，e.g., konsole, dolpin, kate, okular, etc.
$ pacman -S plasma kde-applications sddm
```

## 設置開機啟動服務

```
$ systemctl enable sddm
$ systemctl disable netctl
$ systemctl enable NetworkManager (注意大小寫)
```

## NTFS 檔案系統讀寫支援

Linux kernel 不支援對 NTFS 檔案系統的讀取，如果額外的資料硬碟、其他硬碟是 NTFS 檔案系統的話，想要寫入就必須安裝 `ntfs-3g` Package。

## 其他雜七雜八軟體安裝

懶得寫= =

## 沒有去設定的

Disable Nvidia GPU

## Reference

- [**Installation guide - ArchWiki**](https://wiki.archlinux.org/index.php/Installation_guide)
- [以官方Wiki的方式安装ArchLinux](https://www.viseator.com/2017/05/17/arch_install/)
- [ArchLinux安装后的必须配置与图形界面安装教程](https://www.viseator.com/2017/05/19/arch_setup/)
- [Arch Linux 安装指南[2018.03.01]](https://bbs.archlinuxcn.org/viewtopic.php?id=1037)
- [Arch Linux：安裝筆記](https://blog.rex-tsou.com/2017/12/arch-linux%E5%AE%89%E8%A3%9D%E7%AD%86%E8%A8%98/)
- [Arch Linux 安裝筆記](https://leomao.github.io/2017/09/archlinux-install-note/)
- [给 GNU/Linux 萌新的 Arch Linux 安装指南](https://blog.yoitsu.moe/arch-linux/installing_arch_linux_for_complete_newbies.html)
- [Arch Linux 安裝教學](https://blog.allenchou.cc/arch-linux-tutorial/)
- [geniustanley/arch-linux-install](https://github.com/geniustanley/arch-linux-install)
- [Install Arch Linux on XPS 13 9370](https://gist.github.com/android10/3b36eb4bbb7e990a414ec4126e7f6b3f)