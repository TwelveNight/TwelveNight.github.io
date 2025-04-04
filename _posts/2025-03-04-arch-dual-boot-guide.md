## Arch双系统安装

### 一、下载镜像，制作U盘

- 选择Rufus
- 以dd镜像写入

### 二、开辟linux储存空间

- 磁盘管理
- 开辟大概500G(注意不要分配,随自己喜好)
- 插U盘，进BIOS，从U盘启动

### 三、进入BIOS

- 拯救者重启的时候按住shift不松，开机或重启时显示【Lenovo) logo时，快速连续点按【F2】键
- 将USB优先级调到最高
- 将Secure Boot 调成disable

### 四、正式安装

调节分辨率(可选）@hangone 

```Shell
ctxsetmode = 1024*768
```

先把蜂鸣器关了

```Shell
rmmod pcspkr
```

连接网络

```Shell
rfkill unblock wifi //解锁
iwctl
device list
station [网卡] scan
station [网卡] get-networks //获取network
station [网卡] connect [network]
quit //退出iwctl

//test
ping
```

跟进系统时间

```Shell
timedatectl set-ntp 1
```

硬盘分区

```Shell
fdisk/cfidsk //查看分区
fdisk [硬盘位置] //cd 到需要安装系统的硬盘位置
n //新建分区 默认为提前设置好的未分配空间
w //保存

fdisk -l 查看当前硬盘空间分配 //找到linux filesystem

mkfs.ext4 [linux filesystem所在目录] //创建 ext4 文件系统
mount /dev/nvme8nip9[分区路径] /mnt //挂载分区

//挂在UEFI分区 在前面的EFI system分区
安装linux的时候，efi分区直接挂载win上那个EFI分区就行，因为都是fat32格式，通用的。
mkdir /mnt/boot
mount /dev/nvme8nip1[EFI盘] /mnt/boot
```

设置镜像源

```Shell
vim etc/pacman.d/mirrorlist
```

```Shell
#放在文件最顶端
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

安装基本软件包

```Shell
pacstrap /mnt base base-devel linux linux-firmware linux-headers dhcpcd
```

配置fstap

```Shell
genfstab -L /mnt >> /mnt/etc/fstab //生成并将文件系统表（fstab）的内容追加到指定的文件中
cat /mnt/etc/fstab //确认是否生成成功
```

chroot 切换到已挂载的目录（通常是根文件系统）

```Shell
arch-chroot /mnt //以便在其中进行一些必要的配置步骤，如安装引导加载程序、设置主机名、修改网络配置等。
```

设置时区

```Shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc //将当前系统时间写入硬件时钟
```

安装必备软件

```Shell
pacman -S vim dialog wpa_supplicant ntfs-3g networkmanager netctl
```

设置语言

```Shell
vim /etc/locale.gen
```

```Plain Text
#激活(取消注释)
Zh_CN.UTF-8,zh_HK.UTF-8,zh_TW.UTE-8,
en_US.UTF-8
```

```Shell
//读取系统中的 locale 配置文件，并根据配置文件中定义的区域设置生成对应的 locale 数据库。这些区域设置定义了语言、字符编码、日期格式、货币符号等本地化信息。
locale-gen
```

```Shell
vim /etc/locale.conf
```

```Shell
#设置了系统的默认语言为美式英语（en_US）并使用 UTF-8 字符编码。
LANG=en_US.UTF-8
```

设置主机名

```Shell
vim /etc/hostname
```

```Shell
#主机名
nightArch
```

```Shell
vim /etc/hosts
```

```Shell
127.0.0.1 localhost
::1 localhost
127.0.0.1 nightArch.localdomain nightArch [主机名]
```

设置root密码

```Shell
passwd
```

安装识别其它系统的工具和linux下读写NTFS文件系统(windows系统)的工具

```Shell
pacman -S os-prober ntfs-3g
```

UEFI引导向相关

```Shell
pacman -S grub efibootmgr
```

配置grub

```Shell
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

```Shell
//检查
vim /boot/grub/grub.cfg
```

```Shell
//识别Windows和linux
vim /etc/default/grub
```

```Shell
#取消注释
GRUB_DISABLE_OS_PROBER=false
```

退出挂载目录，重启

```Shell
exit
reboot
```

拔掉U盘

检查是否可以识别到windows系统

### 五、再次进入linux系统进行后续配置

连接网络

```Shell
systemctl enable NetworkManager
systemctl start NetworkManager
nmcli dev wifi
nmcli dev wifi list
nmcli dev wife connect "wifi名" password "..."
```

申请交换空间(虚拟内存的扩展，用于处理系统内存不足的情况)

```Shell
dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
//为交换文件设置权限
chmod 600 /swapfile
//创建交换分区
mkswap /swqpfile
//激活交换分区
swapon /swapfile
```

```Shell
//在系统启动时自动挂载交换分区。
vim /etc/fstab
```

```Shell
#是在装系统的盘的下面加
/swapfile none swap defaults 0 0
```

创建用户

```Shell
useradd -m -G wheel night
passwd night
```

配置sudo

```Shell
pacman -S sudo
ln -s /usr/bin/vim /usr/bin/vi //输入 "vi" 命令时，实际上会执行 "vim" 编辑器
visudo
```

```Shell
#查找激活
wheel ALL=(ALL)ALL
#记得设个sudo免密
...
```

### 六、配置图形化界面

增加储存库

```Shell
su night
```

```Shell
sudo vim /etc/pacman.conf
```

```Shell
#去掉注释
[multilib]
Include = /etc/pacman.d/mirrorlist
#新增
[archlinuxcn]
Server = https://repo.archlinuxcn.org/$arch
```

更新软件源

```Shell
sudo pacman -Syy
```

下载并安装 Arch Linux CN 存储库的密钥环软件包，确保从 Arch Linux CN 存储库安装的软件包的完整性和安全性

```Shell
sudo pacman -S archlinuxcn-keyring
```

安装显卡(根据自身显卡安装)

```Shell
sudo pacman -S xf86-video-nouveau mesa
```

安装kde

```Shell
sudo pacman -S xorg plasma kde-applications sddm network-manager-applet
```

启动服务

```Shell
sudo systemctl enable sddm
sudo systemctl disable netctl
sudo systemctl enable NetworkManager
```

安装中文字体

```Shell
sudo pacman -S wqy-microhei wqy-microhei-lite wqy-bitmapfont wqy-zenhei ttf-arphic-ukai ttf-arphic-uming adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts noto-fonts-cjk
```

安装git

```Shell
sudo pacman -S git
```

安装yay软件包

```Shell
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

重新修改字体并重启

```Shell
sudo vim /etc/locale.conf
```

```Shell
#更新
LANG=zh_CN.UTF-8
```

```Shell
sudo reboot
```

### 七、重新进入kde桌面

Ctrl + alt + T 呼出终端

联网

```Shell
sudo dhcpcd
```

注意有些软件包需要重启才能够生效

### 八、连接蓝牙

```Shell
sudo pacman -S bluez bluez-utils pulseaudio-bluetooth pipewire pipewire-audio
sudo systemctl enable --now bluetooth
sudo usermode -aG lp [用户名]
```

### 九、下载输入法

[Fcitx5 官方文档](https://wiki.archlinux.org/index.php/Fcitx5_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

```Shell
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-anthy # 日文输入引擎
sudo pacman -S fcitx5-pinyin-moegirl # 萌娘百科词库。二刺猿必备（archlinuxcn）
sudo pacman -S fcitx5-material-color # 输入法主题
```

### 十、调节屏幕亮度

```Shell
sudo pacman -S xorg-xbacklight
```

### 十一、NVIDIA显卡驱动修复

[修复NVIDIA显卡驱动](https://bbs.archlinux.org/viewtopic.php?id=285985)

### 十二、emoji缺失

[emoji缺失解决方案](https://szclsya.me/zh-cn/posts/fonts/linux-config-guide/)

### 十三、美化

gurb启动界面

自行探索
