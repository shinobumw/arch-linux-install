---
title: ArchLinux 重灌記錄
description:
tags: linux, note
---

# ArchLinux 重灌記錄

[![hackmd-github-sync-badge](https://hackmd.io/rfZJPmMyRPudCVXRLtFpFQ/badge)](https://hackmd.io/rfZJPmMyRPudCVXRLtFpFQ)


[TOC]

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
- `/boot`: 雙系統的話用原本 windows 的 partition，建議大小 260~512 MiB

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

# New Mount point [1]
$ mkdir /mnt/boot
$ mount /dev/sda2 /boot
or
$ mkdir /mnt/efi
$ mount /dev/sda2 /efi
```

[1] [Partitioning - ArchWiki](https://wiki.archlinux.org/index.php/Partitioning#/boot)

## Mirror List

`$ nano /etc/pacman.d/mirrorlist`

把 Taiwan 的 mirror 移到最上面

## 安裝基本系統

`$ pacstrap /mnt  base base-devel`

:::warning  
**Note:** The `base` group has been replaced by a metapackage of the same name in order to minimize the installation size. [1][2]  
:::

需要額外安裝的:

- `linux-lts`, `linux-firmware`
- file system (e.g. `e2fsprogs`, `ntfs-3g`)
- text editors (e.g. `nano`, `vim`)
- software necessary for networking (e.g. `networkmanager`)
- packages for accessing documentation in man and info pages: `man-db`, `man-pages` and `texinfo`.

This is what's in the group but not "in the package":

<details><summary>Details</summary>
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
</details>

#### Reference:

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

### Multiple kernels

例：原本已安裝 `linux`，想再安裝 `linux-lts` [1]

```bash
$ pacman -S linux-lts
# Regenerate grub config
$ grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
$ vim /etc/default/grub

GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
```

[1] https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#Multiple_entries

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
找到`# %wheel ALL=(ALL)ALL`，把 # 字號拿掉。

`$ reboot`

## GUI

```
$ pacman -S xorg [1]
# xf86-video-intel 不一定要安裝 [2]
$ pacman -S xf86-video-intel
# 可選需要的安裝就好，例：konsole, dolpin, kate, okular, etc.
$ pacman -S plasma kde-applications sddm
```
[1] [
Xorg - ArchWiki](https://wiki.archlinux.org/index.php/Xorg#Driver_installation)  
[2] [Intel graphics - ArchWiki](https://wiki.archlinux.org/index.php/Intel_graphics#Installation)

## 設置開機啟動服務

```
$ systemctl enable sddm
$ systemctl disable netctl
$ systemctl enable NetworkManager (注意大小寫)
```

## NTFS 檔案系統讀寫支援

Linux kernel 不支援對 NTFS 檔案系統的讀取，如果像外接硬碟是 NTFS 檔案系統的話，就必須安裝 `ntfs-3g` Package。

## 其他雜七雜八軟體安裝

### 桌面設定

 - [Wallpaper](https://yande.re/post/show?md5=4f42ad6493368a89010ebdcccb708ed5) 很惠<3
 - Dock at bottom: Latte Dock
 - Panel at top: Latte Dock
 - Plasmoids in the panel: 
   - Application Launcher (menu icon)
   - ~~Active  Window Control(icon+title+application menu)~~
   - Window Title Applet
   - Window AppMenu Applet
   - Event Calendar
     - Setting: `'<font color="#feeff9">'dddd, MMMM d'</font>' '<font color="#f7d19b">'h:mm'</font>'`
   - Pager
   - (Color Picker)
   - System Tray
   - ~~Active Window Control(buttons)~~
   - Window Buttons Applet
 - ~~Kwin Script for tiling: Quarter Tiling or Tilting(downloadable from Kwin Scripts settings)~~ -> Encounter some bugs.

### Themes

- [arc-gtk-theme](https://www.archlinux.org/packages/community/any/arc-gtk-theme/)
- [adapta-gtk-theme](https://github.com/adapta-project/adapta-gtk-theme)
- [materia-theme](https://github.com/nana-4/materia-theme)
- [Flat-Plat-Blue Theme](https://github.com/peterychuang/Flat-Plat-Blue)
  > Forked from Materia Theme (formerly Flat-Plat)
- [numix-gtk-theme](https://github.com/numixproject/numix-gtk-theme)
- [paper-icon-theme](https://aur.archlinux.org/packages/paper-icon-theme/)
- [**arc-kde**](https://github.com/PapirusDevelopmentTeam/arc-kde)
  > [目前使用](https://aur.archlinux.org/pkgbase/arc-kde/)
- [adapta-kde](https://github.com/PapirusDevelopmentTeam/adapta-kde)
- [monochrome-kde](https://gitlab.com/pwyde/monochrome-kde)
  > Only for sddm theme
- [**papirus-icon-theme**](https://www.archlinux.org/packages/community/any/papirus-icon-theme/)
  > 目前使用

### Via pacman

#### Font

- noto-fonts noto-fonts-cjk noto-fonts-emoji
  > **Note**: Remember to set chrome/firefox's fonts to CJK.
- adobe-source-han-sans-otc-fonts adobe-source-han-serif-otc-fonts
- adobe-source-code-pro-fonts
- ttf-ubuntu-font-family
- otf-fira-code

#### Others

- firefox-developer-edition
- tlp ─ 省電用
- openssh
- smplayer smplayer-themes
  - youtube-dl smtube
- latte-dock
- ~~plasma5-applets-active-window-control~~
- neofetch
- networkmanager-openvpn
  > 連 VPN 才需要裝
  > 現在 linux 5.6 以上已內建 wireguard 了。
- tmux
- [zsh](https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH) with [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
- [tldr](https://tldr.sh/)
  - [tldr++](https://aur.archlinux.org/packages/tldr%2B%2B/) (go ver. w/ user interaction)
- (qmmp)
- (termite)
- [code](https://www.archlinux.org/packages/community/x86_64/code/)
- htop, [bashtop](https://github.com/aristocratos/bashtop)
- clang, (astyle)
- tree
- pandoc
  > Convert doc format
- [bat](https://github.com/sharkdp/bat#syntax-highlighting) ─ A cat clone
- rsync rclone
- python
  - python-setuptools
  - python-pip
  - tk (when using Matplotlib)
- (discord)
- (telegram-desktop)
- ([peek](https://www.archlinux.org/packages/community/x86_64/peek/))  ─ A simple screen recorder
- [fd](https://github.com/sharkdp/fd) ─ A simple, fast and user-friendly alternative to 'find'
- [ripgrep](https://github.com/BurntSushi/ripgrep)
- [Glances](https://github.com/nicolargo/glances) ─ CLI curses-based monitoring tool
  - [Glances 命令列系統監控工具](https://blog.gtwang.org/linux/glances-cli-curses-based-monitoring-tool/)
- vim neovim
- ([thefuck](https://github.com/nvbn/thefuck))
- [zstd](https://github.com/facebook/zstd) ─ Zstandard, Fast real-time compression algorithm
  - Compresssion: `tar -acf target.tar.zst file1 file2`
  - Decompression: `tar -axf source.tar.zst`
  - [[Reference]](https://news.ycombinator.com/item?id=21958585)
- docker

### Via AUR

- ~~pacaur (unmaintained)~~
- [yay](https://github.com/Jguer/yay) (Yet another Yogurt)  ─ An AUR Helper written in Go
- ~~sublime-text-dev~~
- visual-studio-code-bin
  > **Note**: `code-git` in AUR and `code` in arch official repos are compiled version from github, and this is the microsoft official binary version.
- google-chrome
- downgrade
- [spotify](https://aur.archlinux.org/packages/spotify/)
- ([gotop](https://aur.archlinux.org/packages/gotop/))
- latte-dock related:
  - plasma5-applets-window-title
  - plasma5-applets-window-appmenu
  - plasma5-applets-window-buttons
- [plasma5-applets-eventcalendar](https://aur.archlinux.org/packages/plasma5-applets-eventcalendar/)
- [jetbrains-toolbox](https://aur.archlinux.org/packages/jetbrains-toolbox/)
  - Pycharm
  - IntelliJ IDEA
  - etc.

#### Reference 

- [Awesome Command-Line Tools](https://www.vimfromscratch.com/articles/awesome-command-line-tools/)
- [awesome-shell](https://github.com/alebcay/awesome-shell)
- [awesome-cli-apps](https://github.com/agarrharr/awesome-cli-apps)
- [What’s your favorite CLI tool nobody knows about? - r/linux](https://www.reddit.com/r/linux/comments/b5n1l5/)

## 沒有去設定的

- Disable Nvidia GPU

## References

- [**Installation guide - ArchWiki**](https://wiki.archlinux.org/index.php/Installation_guide)
- [以官方Wiki的方式安装ArchLinux](https://www.viseator.com/2017/05/17/arch_install/)
- [ArchLinux安装后的必须配置与图形界面安装教程](https://www.viseator.com/2017/05/19/arch_setup/)
- [Arch Linux 安装指南[2019.12.01] / Arch Linux 中文论坛](https://bbs.archlinuxcn.org/viewtopic.php?id=1037)
- [Arch Linux：安裝筆記](https://blog.rex-tsou.com/2017/12/arch-linux%E5%AE%89%E8%A3%9D%E7%AD%86%E8%A8%98/)
- [Arch Linux 安裝筆記](https://leomao.github.io/2017/09/archlinux-install-note/)
- [给 GNU/Linux 萌新的 Arch Linux 安装指南](https://blog.yoitsu.moe/arch-linux/installing_arch_linux_for_complete_newbies.html)
- [Arch Linux 安裝教學](https://blog.allenchou.cc/arch-linux-tutorial/)
- [geniustanley/arch-linux-install](https://github.com/geniustanley/arch-linux-install)
- [Install Arch Linux on XPS 13 9370](https://gist.github.com/android10/3b36eb4bbb7e990a414ec4126e7f6b3f)
- [Arch Linux 安裝筆記 - HackMD](https://hackmd.io/@arthurc0102/HkWytqPLH)