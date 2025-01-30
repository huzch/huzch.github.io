---
title: "Dual Boot Arch & Windows"
date: 2025-01-30
categories:
  - blog
tags:
  - workplace
  - deployment
---

- 用 Rufus 制作 Arch 系统盘

- 用傲梅分区助手：

  - 提前将硬盘 BitLocker 解密
  - 将EFI分区扩大为1GB
  - 创建大小为128G空间（作根目录和家目录）
  - 创建大小为16G空间（作交换分区，大小与内存相同）

- 进入主板 BIOS：

  - 关闭安全启动
  - 调整启动顺序，将U盘启动放在首位

- 插入U盘并重启，进入安装环境

- 禁用 reflector 服务（该服务会自动改变镜像源）

  - ```bash
    systemctl stop reflector.service # 禁用reflector服务
    systemctl status reflector.service # 查看该服务状态，是否被禁用
    ```

- 连接网络

  - ```bash
    iwctl # 进入交互式命令行
    device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
    station wlan0 scan # 扫描网络
    station wlan0 get-networks # 列出所有 wifi 网络
    station wlan0 connect wifi-name # 进行连接，注意这里无法输入中文。回车后输入密码即可
    exit # 连接成功后退出
    ping www.baidu.com # 测试网络连通性
    ```

- 更新系统时钟

  - ```bash
    timedatectl set-ntp true # 将系统时间与网络时间进行同步
    timedatectl status # 检查服务状态
    ```

- 分区

  - ```bash
    lsblk # 显示当前分区情况
    cfdisk /dev/nvmexn1 # 对安装 archlinux 的磁盘分区，x 为数字
    ```

  - 将之前创建的16GB空间类型转换为Linux swap

  - 将之前创建的128GB空间类型转换为Linux filesystem

  - 写入并退出

  - ```bash
    lsblk # 复查磁盘情况
    ```

- 格式化

  - 双系统不用格式化EFI分区

  - ```bash
    mkswap /dev/nvmexn1pn # 格式化 swap 分区
    ```

  - ```bash
    mkfs.btrfs -L myArch /dev/nvmexn1pn # 格式化 btrfs 分区，-L 为设置标签
    ```

  - ```bash
    mount -t btrfs -o compress=zstd /dev/nvmexn1pn /mnt # 为了创建 btrfs 子卷，需先挂载 btrfs 分区，-t 指定挂载分区文件系统类型，-o 添加挂载参数，compress=zstd 为开启透明压缩
    df -h # 复查挂载情况，-h 为以人类可读的方式输出
    ```

  - ```bash
    btrfs subvolume create /mnt/@ # 创建 / 目录子卷
    btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
    btrfs subvolume list -p /mnt # 复查子卷情况
    ```

  - ```bash
    umount /mnt # 创建好子卷后，卸载 btrfs 分区
    ```

- 挂载

  - ```bash
    mount -t btrfs -o subvol=/@,compress=zstd /dev/nvmexn1pn /mnt # 挂载 / 目录
    mkdir /mnt/home # 创建 /home 目录
    mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvmexn1pn /mnt/home # 挂载 /home 目录
    mkdir -p /mnt/boot # 创建 /boot 目录
    mount /dev/nvmexn1pn /mnt/boot # 挂载 /boot 目录
    swapon /dev/nvmexn1pn # 启用交换分区
    df -h # 复查挂载情况
    free -h # 复查交换分区启用情况
    ```

- 更换镜像源

  - ```bash
    vim /etc/pacman.d/mirrorlist # 将距离最近的镜像源放在文件最上面
    ```

  - ```bash
    Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch # 中国科学技术大学开源镜像站
    Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch # 清华大学开源软件镜像站
    Server = https://repo.huaweicloud.com/archlinux/$repo/os/$arch # 华为开源镜像站
    Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch # 兰州大学开源镜像站
    ```

- 安装系统及必要工具

  - ```bash
    pacstrap /mnt base base-devel linux linux-firmware btrfs-progs # 安装基础包
    ```

  - ```bash
    pacstrap /mnt networkmanager vim sudo bash bash-completion # 安装必要工具
    ```

- 生成 fstab 文件（定义磁盘分区，记录挂载情况）

  - ```bash
    genfstab -U /mnt > /mnt/etc/fstab # 生成 fstab 文件
    cat /mnt/etc/fstab # 复查 fstab 文件
    ```

- 从安装环境进入新系统

  - ```bash
    arch-chroot /mnt
    ```

- 设置时区

  - ```bash
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置为中国上海
    hwclock --systohc # 同步硬件时间
    ```

- 本地化

  - ```bash
    vim /etc/locale.gen # 取消 en_US.UTF-8 UTF-8 和 zh_CN.UTF-8 UTF-8 的注释
    ```

  - ```bash
    locale-gen # 生成本地化配置
    ```

  - ```bash
    echo 'LANG=en_US.UTF-8'  > /etc/locale.conf # 设置语言
    ```

- 设置主机名

  - ```bash
    vim /etc/hostname # 比如取名为myArch
    ```

  - ```bash
    vim /etc/hosts # 设置与之匹配的条目，比如加入以下内容：
    127.0.0.1   localhost
    ::1         localhost
    127.0.1.1   myArch.localdomain	myArch
    ```

- 设置root用户密码

  - ```bash
    passwd root
    ```

- 安装微码

  - ```bash
    pacman -S intel-ucode # Intel
    ```

  - ```bash
    pacman -S amd-ucode # AMD
    ```

- 安装引导程序

  - ```bash
    pacman -S grub efibootmgr os-prober # 安装相应软件包
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB # 将GRUB 安装到 EFI 分区
    ```

  - ```bash
    vim /etc/default/grub # 修改 GRUB 配置
    # 1.找到 GRUB_CMDLINE_LINUX_DEFAULT，将其赋值为"loglevel=5 nowatchdog"
    # 2.在文件结尾加上 GRUB_DISABLE_OS_PROBER=false
    ```

  - 注意：nowatchdog 参数无法禁用英特尔的看门狗硬件，需改为 modprobe.blacklist=iTCO_wdt

  - ```bash
    grub-mkconfig -o /boot/grub/grub.cfg # 生成 GRUB 配置
    ```

- 完成安装

  - ```bash
    exit # 退回安装环境
    umount -R /mnt # 卸载新分区
    reboot # 重启
    ```
  
  - 重启时记得拔掉U盘
