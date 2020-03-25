---
title: Arch Linux + Windows 10 雙系統安裝筆記
description:
tags: linux, note
---

# Arch Linux + Windows 10 雙系統安裝筆記

## TOC

[TOC]

## 設備環境

- ASUS UX430UQ
    - i5 7200U
    - 8GB Ram
    - 512GB SSD
    - Windows 10 Home 64bits
- DE: KDE Plasma
    > WM: i3-gaps 等有空再來研究

## 事前準備

- 在電源計劃設定關閉快速啟動
    > windows 每次大更新後都會預設勾選，要記得關掉
- 進 BIOS 關閉 secure boot, fast boot
- 壓縮硬碟空間 (100GB)
- 用 [rufus](http://rufus.akeo.ie/?locale) 製作開機USB

## 安裝過程

[ArchLinux 重灌記錄](https://hackmd.io/rfZJPmMyRPudCVXRLtFpFQ)

### Partition

- /swap - 512MB(8GB以上可不用割)
- /root - remaining(99.5GB)
- /boot - 不用另外割，只要掛載 windows 原本的 EFI System 到 /efi 就行，記得不要不小心 format 掉

[[Reference](https://wiki.archlinux.org/index.php]/EFI_system_partition#Mount_the_partition)]

### 雙系統引導

#### 法一

:::spoiler  略
安裝完開機選單只看到 Archlinux 而沒有 Windows 10，這是正常情况，因為我們沒有寫 windows 的 menu，grub 只能引導進入 archlinux。

先輸入下面兩個指令，把兩個輸出結果先記下來。[註一]

1. $fs_uuid
`# sudo grub-probe --target=fs_uuid /boot/efi/EFI/Microsoft/Boot/bootmgfw.efi`

1. $hints_sting
`# sudo grub-probe --target=hints_string /boot/efi/EFI/Microsoft/Boot/bootmgfw.efi`

編輯/boot/grub/grub.cfg，找到
```
### BEGIN /etc/grub.d/10_linux ###
...
### END /etc/grub.d/10_linux ###
```
在之間底部加上下面這段
```bash
menuentry "Microsoft Windows 10 x86_64 UEFI-GPT" {  
    insmod part_gpt  
    insmod fat  
    insmod search_fs_uuid  
    insmod chain  
    search --fs-uuid --set=root \$hints_string \$fs_uuid
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi  
}
```
这里注意替换下\$hints_string为你之前生成的hints_string，\$fs_uuid为你之前生成的fs_uuid。如果想加，还可以加入shutdown和restart项
```shell
menuentry "System shutdown" {  
    echo "System shutting down..."  
    halt  
}
menuentry "System restart" {  
    echo "System rebooting..."  
    reboot  
}
```
重啟後選單就會出現Windows 10的選項了。

[註一] 路徑可能不同，請依情況更改。

[[Reference]](http://blog.throneclay.com/2017/01/13/wadual/)
:::

#### 法二 (快速)

重新生成設定檔就好了。
`$ grub-mkconfig -o /boot/grub/grub.cfg`

## 使用設定

### 觸控板設定

#### 法一

:::spoiler  略
可通过修改 synaptics 的配置文件，修改触摸板配置。包括多指敲击、滚动、避免手掌触摸、精确度与快速滚动。

`cp /usr/share/X11/xorg.conf.d/70-synaptics.conf /etc/X11/xorg.conf.d/70-synaptics.conf`
```bash 
#file: /etc/X11/xorg.conf.d/70-synaptics.conf
Section "InputClass"
        Identifier "touchpad catchall"
        Driver "synaptics"
        MatchIsTouchpad "on"
        
        Option "TapButton1" "1"            #单指敲击产生左键事件
        Option "TapButton2" "2"            #双指敲击产生中键事件
        Option "TapButton3" "3"            #三指敲击产生右键事件
        
        Option "VertEdgeScroll" "on"       #滚动操作：横向、纵向、环形
        Option "VertTwoFingerScroll" "on"
        Option "HorizEdgeScroll" "on"
        Option "HorizTwoFingerScroll" "on"
        Option "CircularScrolling" "on"  
        Option "CircScrollTrigger" "2"
        
        Option "EmulateTwoFingerMinZ" "40" #精确度
        Option "EmulateTwoFingerMinW" "8"
        Option "CoastingSpeed" "20"        #触发快速滚动的滚动速度
        
        Option "PalmDetect" "1"            #避免手掌触发触摸板
        Option "PalmMinWidth" "3"          #认定为手掌的最小宽度
        Option "PalmMinZ" "200"            #认定为手掌的最小压力值
EndSection
:::

#### 法二 (快速)

直接到 System Settings 的 Touchpad 去設定。

### 輸入法設定

`$ sudo pacman -S fcitx-im kcm-fcitx fcitx-rime fcitx-mozc`

- `kcm-fcitx`: KDE 專用，GTK 則是裝 `fcitx-configtool`
- `fcitx-rime`: 拼音
- `fcitx-mozc`: 日文
- ~~`fcitx-table-extra`: 嘸蝦米~~
- 改用神人修改過的 [rime-liur](https://github.com/hsuanyi-chou/rime-liur)

:::info
**安裝方法**：
1. 下載 [https://github.com/shinobumw/rime-liur](https://github.com/shinobumw/rime-liur)
2. 將`default.custom.yaml`和`liur*.yaml`放到`rime`資料夾底下
3. 重新 deploy rime

**新增功能**：
:::spoiler
1. **注音模式：** 以「`';`」鍵引導可進行注音輸入(但無法透過數字鍵選字)
2. **拼音模式：** 以「\`」鍵(上排數字鍵 1 左邊)引導可進行拼音輸入
3. **讀音反查：** 以「`;;`」鍵引導並輸入無蝦米碼，可反查該字讀音，如「龘」=`ㄉㄚˊ`
4. **日文漢字/罕用字輸入功能**
    1. 字典檔包含日文漢字如「辻」「雫」「渋」等…。不需要的人，可至`liur.extended`中，把`- liur_Japan`這行註解掉
    2. 可透過`ctrl + /`切換至擴充字集，輸入罕用字，如四個金(AAAA)等字
5. **複合型查碼：**
    1. 於造詞、拼音、注音模式下鍵入`ctrl + '`(Enter鍵左邊)，可直接查詢蝦米編碼，
    2. 於注音或拼音模式下，可以進行以詞查碼
        (例：於注音模式下輸入ㄍㄢㄍㄚ或拼音模式下輸入ganga，再切換查碼就可以
        找出「尷尬」這個詞的所有蝦米編碼，減少選字)
        於造詞模式下，則可以反查出該字及其候選字的所有編碼(例：輸miep，可以查
        出微、徵、徽、徾、鰴、徴)等字的蝦米編碼
6. **簡繁轉換：** 於任何模式下透過`ctrl + .`，可進行即時簡繁轉換，無須切換模式
7. **中英混輸：** 在不切換輸入法的情形下，可以空白鍵上中文字或中文符號；Enter 鍵上英文字或英文符號
8. **shift切換中英輸入：** shift 切換中英輸入，caps lock 變為大寫
:::

如果在 firefox 中無法打中文，請在 `~/.xprofile` 中加入:
```bash
# export XIM_PROGRAM=fcitx // 此行非必要
# export XIM=fcitx // 此行非必要
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"

or

echo 'export GTK\_IM\_MODULE=fcitx' >> /home/\[用戶名\]/.xprofile 
echo 'export QT\_IM\_MODULE=fcitx' >> /home/\[用戶名\]/.xprofile 
echo 'export XMODIFIERS=@im=fcitx' >> /home/\[用戶名\]/.xprofile 
chown \[用戶名\]:\[用戶名\] /home/\[用戶名\]/.xprofile
```
[[Reference1](https://github.com/FZUG/repo/issues/33)]  
[[Reference2](https://zijung.me/archives/archlinux-install-record.html)]

## Miscellaneous

### VPN 快速架設 

- [Nyr/openvpn-install](https://github.com/Nyr/openvpn-install)
  > **Note**: Only work on Ubuntu, Debian, and CentOS

  1. `$ wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh`
  2. Follow the instructions indicated.
  3. Copy .ovpn file to your computer.

- [angristan/openvpn-install](https://github.com/angristan/openvpn-install)
  >Another script for archlinux and other distros

- [**Outline VPN**](https://getoutline.org/en/home)

### Downgrading packages

[Downgrading packages #Arch Linux Archive - ArchWiki](https://wiki.archlinux.org/index.php/Downgrading_packages#Arch_Linux_Archive)

## Command

### pacman / yay 常用指令

- Update: `pacman -Syu`
- Completely remove: `pacman -Rsn`
:::danger
-dd 別亂用，會 skip dependency checks
:::

- Clear cache: `paccache -r`
[[Ref.](https://www.maketecheasier.com/clear-package-cache-arch-linux/)]

### 更多 `pacman` 的詳細指令請參考 ArchWiki

- [pacman - ArchWiki](https://wiki.archlinux.org/index.php/pacman)
- [pacman/Tips and tricks](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks)

## 疑難雜症

Q: AUR 軟體安裝後 Application Menu 沒更新?

A:
<法一> `$ kbuildsycoca4` or `$ kbuildsycoca4`

<法二> (沒試過) [[Source](https://bbs.archlinux.org/viewtopic.php?id=64514)]
```
I had the same bug. After fixing it I thought leaving a short how-to may help one or two others as well.
Description of my KDE BUG: 
1. The K-Menu doesn't get updated even when using the "save" button. Restarting doesn't effect said problem.
2. The K-Menu update command: 'kbuildsycoca4' returns an error (although it doesn't give you any clew as to the originating file of the error).
Solution that worked for me:
1. Rename the file "$HOME/.kde/share/config/plasma-desktop-appletsrc" to "$HOME/.kde/share/config/plasma-desktop-appletsrc.broken" (just something else in fact)
2. Log off and on again. (KDE will now create a new configuration file for the plasmoids)
3. Execute 'kbuildsycoca4' - it will work now, then execute 'kbuildsycoca4 --noincremental' to let it repair the buggy sections in the rest of KDE as well
4. Add some plasmoids, change the wallpaper - just something for KDE4 to process and change its config files.
5. Change the TTY while being logged in (Ctrl+Alt+F1 - just any other TTY will do to get access to your home directory .kde directory)
Note: Some basic shell commands are needed: "cd", "mv", "ls (with the -a for hidden files and directories)" - look them up if you don't feel confident using the shell.
6. Rename the broken file ("$HOME/.kde/share/config/plasma-desktop-appletsrc.broken") to its original name ("$HOME/.kde/share/config/plasma-desktop-appletsrc") and log off from the shell (in the new TTY)
7. Change back to the still logged in KDE4 TTY (try all Ctrl+Alt+F1-12 keys if you don't know which TTY it was), then log off.
8. Login again and you will see your once broken KDE4 with it's complete config - run 'kbuildsycoca4' it should complete without errors, then run a 'kbuildsycoca4 --noincremental' to make sure.
9. Done - your KDE4 K-Menu among other things should work again.
Note that you do not need to use the old configuration file because the file "$HOME/.kde/share/config/plasma-desktop-appletsrc" only contains the plasmoid configuration (selected wallpaper, panels, just the plasmoid-desktop stuff). Everything else will be just as you left it (hotkeys, installed plasmoids, downloaded wallpapers, dolphin configurations, ...).
If there is someone where this fix does NOT work even after completing everything and googling for a bit - leave a note. I'll try to help you when I have the time.
After this bug I decided to make a weekly "$HOME/.kde" directory backup
```

---

Q: Windows 10 時間不準確?\
A: [[Reference](https://wiki.archlinux.org/index.php/Time#UTC_in_Windows)]

1. 關閉網路對時
2. Using regedit, add a DWORD value with hexadecimal value 1 to the registry:
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation\RealTimeIsUniversal`

---

Q: Windows 更新後開機不會出現 Grub 選單?\
A: 進 BIOS 把第一順位的的 WindowsBootLoader 改掉。

---

Q: No user icon on login session?\
A: 權限問題，請參考 [Readme](https://github.com/sgerbino/sddm)。

---

Q: Can I use my super key to open the latte dock app launcher?

A: [[Reference](https://github.com/psifidotos/Latte-Dock/wiki/F.A.Q.)]
```
kwriteconfig5 --file ~/.config/kwinrc --group ModifierOnlyShortcuts --key Meta"org.kde.lattedock,/Latte,org.kde.LatteDock,activateLauncherMenu"
qdbus org.kde.KWin /KWin reconfigure
```

---

Q: 安裝 Archlinux 時用 USB 開機卡在 rootfs?\
A: 用 rufus 製作開機 USB 時選 DD，不要選 ISO。

---

Q: 每次開機連 wifi 都要重新輸入密碼?\
A: 去設定中刪掉後再重連即可。

---

Q: Windows10 大更新後開機沒出現 grub 選單，反而進入 grub rescue？

A: [[Reference1](http://jeffyon.blogspot.tw/2016/08/windows-10-ubuntu-grub-rescue.html?m=1)], [[Reference2](http://roachsinai.github.io/2016/10/14/1arch_install/)]
- 查看有哪些硬碟分區
`# ls` 
- 慢慢找哪個是開機碟（原本是gpt6，windows更新後新增了一個修復碟）
`# ls (hd0,gpt7)/boot` 
- 查看
`# set` 
- 更改（通常是最後一個）
`# set prefix=(hd0,gpt7)/boot/grub`
`# set root=hd0,gpt7`
- 返回normal mode就能看到熟悉的grub選單
`# insmod normal`  
`# normal`
- 進系統後重新安裝grub
`# sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub --recheck`
- 生成配置文件
`# sudo grub-mkconfig -o /boot/grub/grub.cfg`
- 重開機，大功告成！

---

Q：執行`＄pacman -Syu`更新時出現下列訊息該如何處理？
> :: [PackageName] is not present in AUR -- skipping"

A：出現此訊息就表示在官方的 repo 和 AUR 中都找不到這個 package，可能是因為 package 改了名字或是被移除掉，解決辦法：
1. `$ pacman -Rcs [PackageName]` 移除掉此 package。
2. 重新安裝改名後的 package，會顯示遇到衝突，問你是否要移除原本舊名字的 package。

---

Q：為什麼在 plasma(KDE) 上用 firefox 瀏覽大部份網站時的英文字型都顯示等寬字型？  
A：`$ sudo pacman -S ttf-dejavu ttf-liberation `

[Ref.1] [[SOLVED] How to fix the terrible font in firefox in KDE?](https://bbs.archlinux.org/viewtopic.php?id=231893)  
[Ref.2] https://wiki.archlinux.org/index.php/KDE#Fonts

---

Q：How to solve “error: failed to commit transaction (conflicting files)”？  
A：`$ sudo pacman -Syu --overwrite [filepath]`

[Ref.1] [How To Solve “error: failed to commit transaction (conflicting files)” In Arch Linux](https://www.ostechnix.com/how-to-solve-error-failed-to-commit-transaction-conflicting-files-in-arch-linux/)

---

Q：clang-format failed on VSCode?
Error message:
```
Formatting failed:
/home/wctsai/.vscode/extensions/ms-vscode.cpptools-0.21.0/bin/../LLVM/bin/clang-format -style=file -fallback-style=LLVM -assume-filename=/home/wctsai/Documents/github/OJ/LeetCode/x.cpp
  /home/wctsai/.vscode/extensions/ms-vscode.cpptools-0.21.0/bin/../LLVM/bin/clang-format: error while loading shared libraries: libtinfo.so.5: cannot open shared object file: No such file or directory
```

A：`＄ sudo ln -sf /usr/lib/libncursesw.so.6 /usr/lib/libtinfo.so.5`

[Ref.1] [error while loading shared libraries: libtinfo.so.5](https://github.com/commercialhaskell/stack/issues/1012#issuecomment-387054798)

