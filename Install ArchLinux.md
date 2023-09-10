# Install ArchLinux

1. Download Arch Linux ISO

    [archlinux-x86_64.iso](https://archlinux.org/download/)

2. U 盘 ventoy 准备

    略

    选择 `Arch Linux install medium (x86_64, UEFI)` 启动安装环境

    进入 `root@archiso` 后，需要设置互联网，推荐使用网线连接

    > 检查网络接口是否已经启用
    >
    > `ip link`
    >
    > `2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...`
    >
    > 尖括号内的“UP”，表示接口已经启用，否则使用以下命令：`ip link set enp0s3 up`
    >
    > 请使用 ping 命令测试网络: `ping www.baidu.com`
    >

3. 更新系统时钟: 在互联网连接之后，systemd-timesyncd 服务将自动校准系统时间，便于安装软件包时验证签名

    ```bash
    timedatectl
    ```

4. 分区设置

    ```bash
    mkfs.ext4 /dev/nvme1n1p7 #用作根分区，挂载到 /
    # mkfs.fat -F32 /dev/nvme1n1p3 #用作EFI分区 ，挂载到 /boot/efi
    # 如果安装Windows时已经有个EFI分区，就把上面的/dev/sda1换成已有的EFI分区
    mkfs.ext4 /dev/nvme1n1p8 # 挂载到 /home 目录


    mount /dev/nvme1n1p7 /mnt #挂载根目录

    mkdir -p /mnt/boot/efi #EFI分区的挂载点
    mount /dev/nvme1n1p3 /mnt/boot/efi #挂载EFI分区

    mount --mkdir /dev/nvme1n1p8 /mnt/home
    ```

5. 选择软件镜像仓库

    - 方式 1

    ```bash
    reflector -p https -c China --delay 3 --completion-percent 95 --sort rate --save /etc/pacman.d/mirrorlist
    ```

    - 方式二

    手动修改`/etc/pacman.d/mirrorlist`

    ```bash
    vim /etc/pacman.d/mirrorlist
    ------------------------
    Server = https://mirrors.bfsu.edu.cn/archlinux/$repo/os/$arch
    Server = https://mirrors.sjtug.sjtu.edu.cn/archlinux/$repo/os/$arch
    Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
    Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
    ```

6. 安装基础包

    ```bash
    pacstrap /mnt bash base base-devel linux linux-firmware linux-headers vim

    // fstab
    genfstab -U -p /mnt >> /mnt/etc/fstab
    ```

7. chroot

    ```bash
    arch-chroot /mnt

    # 时区
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    hwclock --systohc

    # hostname
    vim /etc/hostname
    ---
    键入：`arch`
    ---

    # 设置 locale
    vim /etc/locale.conf
    ---
    键入：`LANG_en_US.UTF-8`
    ---

    vim /etc/locale.gen
    ---
    取消注释：`#en_US.UTF-8 UTF-8`
    取消注释：`#zh_CN.UTF-8 UTF-8`
    ---
    locale-gen
    
    # 网络管理器
    pacman -S networkmanager
    systemctl enable NetworkManager.service

    # root password
    passwd
    ---
    键入密码：xxxxxx
    ---

    # ucode
    cat /proc/cpuinfo | grep "model name"
    pacman -S intel-ucode # amd-ucode

    # 安装引导加载程序
    pacman -S grub efibootmgr os-prober
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
    # 配置 os-prober
    vim /etc/default/grub
    ---
    取消注释:`GRUB_DISABLE_OS_PROBER=false`
    ---
    grub-mkconfig -o /boot/grub/grub.cfg

    sudo pacman -S plasma sddm     
    sudo systemctl enable sddm
    
    # reboot
    exit
    swapoff /mnt/swapfile
    umount -R /mnt
    reboot
    ```

---

1. Create user and usergroup

    ```bash
        useradd -m -G wheel tianen
        passwd tianen
        ---
        键入密码
        ---

        # 修改权限
        pacman -S sudo man-pages man-db
        vim /etc/sudoers
        ---
        取消注释：`%wheel ALL=(ALL:ALL) ALL`
        ---
        su - tianen
    ```

2. pacman 配置

    ```bash
    sudo vim /etc/pacman.conf
    ----------------
    # Misc options
    Color
    ParallelDownloads = 5
    ---
    [multilib]
    Include = /etc/pacman.d/mirrorlist
    ---
    键入：
    [archlinuxcn]
    Server = https://mirrors.bfsu.edu.cn/archlinuxcn/$arch
    ---

    sudo pacman -Syyu
    sudo pacman -S archlinuxcn-keyring
    ```
