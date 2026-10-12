---
type: 原子
status: 整理中
mastery: 2
tags:
  - 技术/Linux
created: "2026-06-15"
updated: "2026-06-15"
up: "[[00_开发技能MOC]]"
---

# 📅2026-06-14

下面是为你整理好的 **Arch Linux 完整安装与配置脚本文档**，所有命令按顺序整理，你可以直接复制到文本文件保存，后续安装时按步骤执行即可。

---

## Arch Linux 完整安装与配置脚本（可直接复制）

### 一、Live 环境安装阶段（从ISO启动后执行）

```bash
# 1. 确认磁盘
fdisk -l

# 2. 分区（GPT格式，示例为/dev/sda）

# 分区建议：

# /dev/sda1 512MB EFI分区

# /dev/sda2 4GB Swap分区

# /dev/sda3 剩余空间 根分区

# 3. 格式化分区
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2
mkfs.ext4 /dev/sda3

# 4. 挂载分区
mount /dev/sda3 /mnt
mount /dev/sda1 /mnt/boot/efi
swapon /dev/sda2

# 5. 配置国内镜像源
cat > /etc/pacman.d/mirrorlist <<EOF
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/\$repo/os/\$arch
Server = https://mirrors.ustc.edu.cn/archlinux/\$repo/os/\$arch
EOF

# 6. 安装基础系统
pacstrap /mnt base linux linux-firmware vim sudo

# 7. 生成fstab
genfstab -U /mnt >> /mnt/etc/fstab

# 8. 进入chroot环境
arch-chroot /mnt
```

---

### 二、chroot 环境配置阶段

```bash
# 1. 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc

# 2. 本地化配置
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# 3. 设置主机名
echo "archum" > /etc/hostname

# 4. 设置root密码
passwd

# 5. 安装GRUB引导（UEFI模式）
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

# 6. 退出chroot并重启
exit
umount -R /mnt
reboot
```

---

### 三、首次启动后网络修复阶段（重点踩坑部分）

```bash
# 1. 以root用户登录后，启用网卡（以ens33为例）
ip link set ens33 up

# 2. 配置systemd-networkd
cat > /etc/systemd/network/20-wired.network <<EOF
[Match]
Name=ens33

[Network]
DHCP=yes
DNS=223.5.5.5
DNS=114.114.114.114
EOF

# 3. 启用网络服务
systemctl enable systemd-networkd
systemctl restart systemd-networkd
systemctl enable systemd-resolved
systemctl start systemd-resolved
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# 4. 强制配置DNS并锁定文件
echo "nameserver 223.5.5.5" > /etc/resolv.conf
echo "nameserver 114.114.114.114" >> /etc/resolv.conf
chattr +i /etc/resolv.conf

# 5. 验证网络
ping -c 4 www.baidu.com
ping -c 4 mirrors.tuna.tsinghua.edu.cn
```

---

### 四、系统基础配置阶段

```bash
# 1. 更新系统
pacman -Syu

# 2. 创建普通用户（替换为你的用户名，如wangbohan）
useradd -m -G wheel -s /bin/bash wangbohan
passwd wangbohan

# 3. 配置sudo权限
EDITOR=vim visudo
# 找到并取消注释：%wheel ALL=(ALL:ALL) ALL
```

---

### 五、桌面环境（GNOME）安装阶段

```bash
# 1. 安装GNOME及组件
pacman -S gnome gnome-tweaks
# 安装过程中，所有依赖选择直接按回车（默认即可）

# 2. 启用GDM显示管理器
systemctl enable gdm

# 3. 安装VMware增强工具（虚拟机必备）
pacman -S open-vm-tools
systemctl enable vmtoolsd
systemctl start vmtoolsd

# 4. 重启进入桌面
reboot
```

---


