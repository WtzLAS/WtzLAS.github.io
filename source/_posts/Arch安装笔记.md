---
title: Arch安装笔记
date: 2021-10-30 20:06:07
tags:
  - Arch
  - Linux
  - 安装
---

# 准备

这部分的工作与ArchWiki上的[Installation Guide第一部分](https://wiki.archlinux.org/title/Installation_guide#Pre-installation)大同小异，熟悉的读者大概可以跳过（？:thinking:）。

## 准备ArchISO环境

由于众所周知的原因，最好从镜像站下载ISO文件。这里推荐从[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/)下载。

下载完成后，Windows下用[Rufus](https://rufus.ie/)刻录到U盘中。

Linux下使用dd写入。

```
# dd bs=4M if=path/to/archlinux-version-x86_64.iso of=/dev/sdx conv=fsync oflag=direct status=progress
```

然后启动到ArchISO环境即可。

## 联网

如果是有线连接（使用DHCP）这时候systemd-networkd应该已经搞定了，如果是无线网络还需要使用iwctl连接。

```
# iwctl
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect <YOUR_NETWORK_SSID>
```

连接完成后用ping测试一下确保网络通畅。

## 设置pacman镜像

[reflector](https://wiki.archlinux.org/title/Reflector)是用来自动更新`/etc/pacman.d/mirrorlist`的Python脚本，在网络上线后会立刻由systemd执行，但是默认的配置过于铸币:angry:，因此联网之后直接先关闭它。然后执行我们优化过的方案。

```
# systemctl stop reflector
# reflector -c china -f 5 --sort score --save /etc/pacman.d/mirrorlist
```

## 设置pacman

修改`/etc/pacman.conf`，取消`Color`和`ParallelDownload=5`前的注释，这样你的pacman就有色彩和并行下载了。

## 更新时钟

没啥好说的，更就完事了。

```
# timedatectl set-ntp true
```

## 分区，格式化和初步挂载

参考布局：

| 挂载点                        | 分区             | 分区类型                                                     | 文件系统类型 | 建议大小 |
| ----------------------------- | ---------------- | ------------------------------------------------------------ | ------------ | :------- |
| `/mnt/boot`                   | `/dev/esp`       | [EFI system partition](https://en.wikipedia.org/wiki/EFI_system_partition) | FAT32        | 512MiB   |
| `/mnt`,`/mnt/swap`,`/mnt/var` | `/dev/root_part` | Linux filesystem                                             | btrfs        | 剩余空间 |

随后进行格式化和初步挂载：

```
# mkfs.fat -F 32 /dev/esp
# mkfs.btrfs /dev/root_part
# mount /dev/root_part /mnt
```

如果与Windows装双系统，请务必先安装Windows，然后将Windows建立的ESP直接挂载到`/mnt/boot`，不可再次格式化。

## btrfs subvolume创建和挂载

btrfs subvolume的详细定义可参考[btrfs wiki中的说明](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes)。

关于btrfs的挂载参数和chattr可参考[其manual中的说明](https://man.archlinux.org/man/btrfs.5#MOUNT_OPTIONS)和[ArchWiki中的相关介绍](https://wiki.archlinux.org/title/Btrfs#Disabling_CoW)。

```
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@var
# chattr +C /mnt/@var
# btrfs subvolume create /mnt/@swap
# chattr +C /mnt/@swap
# umount -R /mnt
# mount /dev/root_part /mnt -o subvol=@,autodefrag,compress=zstd,discard=async,space_cache,ssd,noatime
# mkdir /mnt/var
# mount /dev/root_part /mnt/var -o subvol=@var,autodefrag,discard=async,compress=none,space_cache,ssd,noatime
# mkdir /mnt/swap
# mount /dev/root_part /mnt/swap -o subvol=@swap,compress=none,space_cache,ssd,noatime
# mkdir /mnt/boot
# mount /dev/esp /mnt/boot
```

## 建立swapfile

参考[ArchWiki中的相关说明](https://wiki.archlinux.org/title/Btrfs#Swap_file)。

```
# cd /mnt/swap
# touch swapfile
# truncate -s 0 ./swapfile
# chattr +C ./swapfile
# btrfs property set ./swapfile compression none
# dd if=/dev/zero of=./swapfile bs=1M count=<YOUR_INTENTED_SWAP_SPACE> status=progress
# chmod 600 ./swapfile
# mkswap ./swapfile
# swapon ./swapfile
```

# 安装与初步配置

## 安装必要包

| 包名                                    | 备注                                                   |
| --------------------------------------- | ------------------------------------------------------ |
| base                                    | -                                                      |
| base-devel                              | 提供makepkg等使用AUR必要的工具                         |
| linux-zen                               | linux kernel, but optimizied for desktop usage         |
| linux-zen-headers                       | 为使用DKMS做准备                                       |
| linux-firmware                          | -                                                      |
| btrfs-progs                             | 用了btrfs还想不装？                                    |
| nano                                    | 喜欢vim的可以换成vim                                   |
| man-db, man-pages, texinfo              | 必备手册，有时候能救命                                 |
| iwd                                     | 使用无线网络                                           |
| git                                     | -                                                      |
| zsh                                     | 1202年了还有人在除了root之外的地方用bash吗？:thinking: |
| grub, dosfstools, efibootmgr, os-prober | grub2 yyds                                             |

```
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware btrfs-progs nano man-db man-pages texinfo iwd git zsh grub dosfstools efibootmgr os-prober
```

## 生成`/etc/fstab`

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

记得检查下`/etc/fstab`里的挂载选项，有时候会丢。

## 复制无线网络配置

你不会想再来一遍的。

```
# mkdir -p /mnt/var/lib/iwd
# cp /var/lib/iwd/* /mnt/var/lib/iwd
```

## chroot

```
# arch-chroot /mnt
```

## 时区配置

```
# ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# hwclock --systohc
```

## 本地化

修改`/etc/locale.conf` ，取消`en_US.UTF-8 UTF-8`和`zh_CN.UTF-8 UTF-8`的注释。

然后运行locale-gen，设置成英文（推荐在安装图形界面之前使用英文）。

```
# locale-gen
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```

## 配置网络

```
# echo <YOUR_HOSTNAME> > /etc/hostname
```

看起来systemd帮你搞定本地主机名解析啦:laughing:，:thinking:就不用改`/etc/hosts`了。

然后在`/etc/systemd/network/`下新建`20-wired.network`和`25-wireless.network`。

这里只配置了DHCP网络需要的内容，特殊需求请自行查阅ArchWiki。

`20-wired.network`加入以下内容：

```
[Match]
Name=en*

[Network]
DHCP=yes
```

`25-wireless.network`加入以下内容：

```
[Match]
Name=wlan*

[Network]
DHCP=yes
```

随后执行：

```
# systemctl enable iwd
# systemctl enable systemd-networkd
```

## 设置root密码

```
# passwd
```

球球大伙别123456了。再用我就：:facepunch::facepunch::facepunch:

## 安装CPU微码

下面这俩安一个就行了。

除非...:thinking:

你有一台同时安装AMD CPU和Intel CPU的电脑？！

（可移动介质安装除外，这俩你都得装。）

```
# pacman -S amd-ucode intel-ucode
```

## 设置引导

修改`/etc/default/grub`，添加`GRUB_DISABLE_OS_PROBER=false`来启动grub的os-prober功能。

没人会想去手写grub的配置文件的。:cry:

```
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

可移动介质还请加入`--removable --recheck`。

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

## 重启

行了，你的Arch已经能自己动了，记得把安装介质拔出去。（？

```
# reboot
```

## 配置网络（第二部分）

不能忘了systemd-resolved啊！

```
# systemctl enable systemd-resolved
# systemctl start systemd-resolved
# ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## 添加用户

```
# useradd -m -G wheel -s /bin/zsh <USERNAME>
# passwd <USERNAME>
```

## 设置sudoer

```
EDITOR=nano visudo
```

使用vim的请换成`EDTIOR=vim`。

# 安装图形化界面

马上就是最后一步啦~:raised_hands:

鉴于GNOME对Fractional Scaling的支持非常难以言喻，HiDPI屏幕可以战略性放弃GNOME了。

当然是用香香的KDE啦~:laughing:

## 安装显卡驱动

参考[ArchWiki上关于显卡驱动的介绍](https://wiki.archlinux.org/title/Xorg#Driver_installation)。

这里安装xf86-video-amdgpu和nvidia-dkms，（[关于DKMS的介绍](https://wiki.archlinux.org/title/Dynamic_Kernel_Module_Support)）。

```
# sudo pacman -S xf86-video-amdgpu nvidia-dkms libva-mesa-driver mesa-vdpau
```

## 安装字体

起码得有个能顶顶的。

```
# sudo pacman -S noto-fonts-cjk noto-fonts-emoji
```

## 安装KDE和部分应用

精挑细选，把一堆游戏都删掉了。挑几个重点说说。

| 包名      | 备注                       |
| --------- | -------------------------- |
| ark       | WinRAR                     |
| dolphin   | Explorer                   |
| vlc       | PotPlayer                  |
| flameshot | Snipaste                   |
| gwenview  | AcdSee(?)                  |
| kcalc     | Calc, but MUCH MORE shabby |
| kate      | Notepad, but more advanced |
| konsole   | cmd                        |
| okular    | Adobe Acrobat(?)           |
| yakuake   | Y Y D S                    |

```
# sudo pacman -S plasma-meta ark audiocd-kio dolphin dolphin-plugins vlc flameshot filelight gwenview kalarm kcalc kate kcolorchooser kcron kdeconnect kdegraphics-thumbnailers kdenetwork-filesharing kdepim-addons kdesdk-kioslaves kdesdk-thumbnailers kdf kdialog kfind kio-extras kio-gdrive kipi-plugins kmag kmousetool konsole korganizer krfb ksystemlog kwalletmanager okular partitionmanager pim-data-exporter pim-sieve-editor print-manager signon-kwallet-extension yakuake zeroconf-ioslave
```

## 配置SDDM

```
# systemctl enable sddm
```

# 进阶设置

## 安装输入法

这里使用无敌的fcitx5 + rime。

```
# sudo pacman -S fcitx5-im fcitx5-rime
```

修改`/etc/environment`加入以下内容：

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
```

## AUR Helper

我是Rust铁粉所以我用paru。

```
# git clone https://aur.archlinux.org/paru.git
# cd paru
# makepkg -si
# cd ..
# rm -rf paru
```

从此之后就可以丢掉pacman了。（雾）

## 音频设置

[pipewire](https://pipewire.org/)是最新一代的Linux音频技术，用就完事了！

```
# paru -S pipewire pipewire-pulse pipewire-jack pipewire-alsa gst-plugin-pipewire
```

有冲突包一律卸载，pipewire会接替这些包的位置。

## 科学上网

我使用xray + qv2ray的方案进行科学上网。

```
# paru -S xray qv2ray-dev-git proxychains
```

修改`/etc/proxychains.conf`，取消`quiet_mode`的注释，然后修改最后一行的代理地址。

## fcitx5皮肤

```
# paru -S fcitx5-breeze
```

接着在fcitx5的配置工具中切换皮肤。

## 配置zsh

为了让zsh更好看更好用，oh-my-zsh是必需的啦~

先安装一个Nerd Font字体以支持奇怪符号。

```
# paru -S nerd-fonts-meslo
```

然后修改konsole配置使用MesloLGS Nerd Font Mono字体。

这里使用antigen来做zsh的包管理。

```
# paru -S antigen
```

修改`~/.zshrc`，加入以下内容：

```
source /usr/share/zsh/share/antigen.zsh

antigen use oh-my-zsh

antigen bundle sudo
antigen bundle zdharma/fast-syntax-highlighting
antigen bundle zsh-users/zsh-autosuggestions
antigen bundle zsh-users/zsh-completions

antigen theme romkatv/powerlevel10k

antigen apply
```

随后：

```
# proxychains zsh
```

然后你的zsh就有炫酷的语法高亮、自动建议和prompt啦~

## 本地化（第二部分）

如果你发现本地化还有问题，请先在KDE的系统设置里设置，然后执行：

```
sudo localectl set-locale LANG=zh_CN.UTF-8
```

# 尾声

到这里你的Arch已经基本完成，图形化界面都有，开始你的愉快新生活吧~
