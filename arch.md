# Arch Linux Manu

### Install Arch

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

    # mount
    mount /dev/nvme1n1p7 /mnt #挂载根目录
    mkdir -p /mnt/boot/efi #EFI分区的挂载点
    mount /dev/nvme1n1p1 /mnt/boot/efi #挂载EFI分区
    mount --mkdir /dev/nvme1n1p8 /mnt/home
    ```

5. 选择软件镜像仓库

    手动修改`/etc/pacman.d/mirrorlist`

    ```bash
    vim /etc/pacman.d/mirrorlist
    ---
    Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
    Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
    ---

    pacman -Sy archlinuxcn-keyring
    pacman -Syyu 
    ```

6. 安装基础包

    ```bash
    pacstrap /mnt bash base base-devel linux-lts  linux-headers linux-firmware vim

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
    
    # 网络管理器，蓝牙
    pacman -S networkmanager bluez bluez-utils pulseaudio-bluetooth alsa-utils pulseaudio pulseaudio-alsa 
    systemctl enable NetworkManager.service
    systemctl enable bluetooth.service
    
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
    
    # Create user and usergroup
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
    
    # KDE
    sudo pacman -S plasma xorg nvidia-lts dolphin konsole fish noto-fonts-cjk noto-fonts-emoji
    sudo systemctl enable sddm
    
    # reboot
    exit
    swapoff /mnt/swapfile
    umount -R /mnt
    reboot
    ```

### Software

1. Check NetworkManager

    ```bash
    ping baidu.com
    systemctl enable NetworkManager
    ```

2. pacman 镜像修改

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
    Server = https://mirrors.utsc.edu.cn/archlinuxcn/$arch
    ---

    sudo pacman -Syyu
    sudo pacman -S archlinuxcn-keyring
    ```

3. postInstall

    ```bash
    sudo pacman -S yay
    pacman -Sc # 清除软件缓存，即/var/cache/pacman/pkg目录下的文件
    pacman -Rns package_name
    pacman -U pacage.tar.zx # 从本地文件安装
    pactree pacage_name # 显示软件的依赖树

    yay -S fish
    # curl -L https://get.oh-my.fish | fish 
    fish_config

    sudo vim /etc/systemd/system/clash.service
    sudo systemctl daemon-reload 
    sudo systemctl enable clash 
    sudo systemctl start clash 
    sudo systemctl status clash

    sudo pacman -S obs-studio
    ```

    ```bash
    sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons fcitx5-material-color

    # Create .xprofile and key `export QT_IM_MODULE=fcitx5`
    vim ~/.xprofile 
    export QT_IM_MODULE=fcitx5

    # .pam_environment
    GTK_IM_MODULE DEFAULT=fcitx
    QT_IM_MODULE  DEFAULT=fcitx
    XMODIFIERS    DEFAULT=\@im=fcitx
    SDL_IM_MODULE DEFAULT=fcitx
    ```

    ```bash
    yay -S clash-for-windows-bin 

    yay -Sy neofetch google-chrome obs-studio baidunetdisk nutstore-experimental xunlei-bin telegram-desktop libreoffice-still libreoffice-still-zh-cn gitkraken visual-studio-code-bin typora-free redis net-tools pot-translation translate-shell okular spectacle gwenview kcalc wemeet-bin vlc zy-player-bin wget ark shotcut inkscape ninja gnu-netcat tcpdump
    
    yay -S electronic-wechat-uos-bin linuxqq lx-music-desktop-appimage
    ```

4. 优化

    **TRIM**

    TRIM会帮助清理SSD中的块，从而延长SSD的使用寿命

    ```bash
    sudo systemctl enable fstrim.timer
    sudo systemctl start fstrim.timer
    ```

---

### git

#### 配置git

1. 设置`user.name`和`user.emal`

   ```bash
   git config --global user.name "lutianen"
   git config --global user.email 753766396@qq.com
   
   # check
   git config --list
   ```

2. 生成密钥

   ```bash
   ssh-keygen.exe -t rsa -C '753766396@qq.com'
   ```

   > 上述代码执行完成后，要求多次输入密码，**请不要输入密码**

3. github配置 SSH Keys

   1. 打开生成的 `Key` 文件 `/.ssh/id_rsa.pub`

   2. 复制全部内容，在 Key 中粘贴

      ![image-20211009162316486](C:\Users\957\AppData\Roaming\Typora\typora-user-images\image-20211009162316486.png)

      ---

#### git 常用命令

- `git status`
- `git clone`
- `git pull`
- `git push`
- `git commit -m 'commits'` or `git commit -m 'commits' xxx.fileType`
- `git add .` or `git xxx.fileType`
- `git reflog`

---

#### Git实现从本地添加项目到远程仓库

> Steps:
>
> 1. 创建一个新的远程仓库 - `Create a new repo` `Create repository`
> 2. 创建并初始化本地仓库 - `git init`
>    - 可添加待上传到远程仓库的项目文件
> 3. 远程仓库和本地仓库关联 - `git remote add origin git@github.com:lutianen/System4CE7`
> 4. 项目文件添加、提交、推送
>    - `git add file`
>    - `git commit -m 'commit statements' file`
>    - `git push -u origin master` 
>      - *由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来
>      - *在以后的推送或者拉取时就可以简化命令*

### Present Windows

<img src="https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202309141103383.png" alt="image-20230914110307330" style="zoom:67%;" />

### scp 

**文件上传、下载**

- 上传 `scp .\cifar-10-python.tar.gz lutianen@10.170.46.236:/home/lutianen/`
- 下载  `scp root@100.100.100.100:/var/tmp/a.txt /var`

### picgo 配置

**安装 picgo**

```bash
yay -S picgo
```

**picgo 配置 github**

- github 获取 **token**

  ![](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/Screenshot_20230912_221106.png))

- PicGo 配置

  ![image-20230912222947951](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/image-20230912222947951.png)

  - 设定仓库名：上文在 GitHub 创建的仓库 `lutianen/PicBed`
  - 设定分支名：`master`
  - 设定 Token：上文生成的 token
  - 指定存储路径：为空的话会上传到跟目录，也可以指定路径
  - 设定自定义域名：可以为空，这里为了使用 CDN 加快图片的访问速度，按这样格式填写：[https://cdn.jsdelivr.net/gh/lutianen/PicBed/@master](https://cdn.jsdelivr.net/gh/lutianen/PicBed/@master)

---

### **SWAP 设置**

<https://wiki.archlinux.org/title/Swap#Swappiness>

- 查看 swap 使用率，一般是 60 ，意思是 60% 的概率将内存整理到 swap

    ```bash
    cat /proc/sys/vm/swappiness
    ```

- 修改 swap 使用策略为 10%，即 10% 的概率将内存整理到 swap

    ```bash
    sudo sysctl -w vm.swappiness=10
    ```

- 修改配置文件：`sudo vim /etc/sysctl.d/99-swappiness.conf` 在文件末尾加上下面这行内容：`vm.swappiness=10`

- 重启后可查看 swappiness 的值

    ![image-20230723115427188](https://raw.githubusercontent.com/lutianen/PicBed/main/202307231154321.png)

---

**Systemd journal size limit**

控制日志最大可使用多少磁盘空间，修改`/etc/systemd/journald.conf` 中的`SystemMaxUse`参数 `SystemMaxUse=50M`

---

### **内核更换**

- Install The Desired Kernel

    `sudo pacman -S linux-lts linux-lts-headers`

- Editing GRUB Config File

    ```bash
    sudo vim /etc/default/grub

    # ---
    `GRUB_DISABLE_SUBMENU=y`    # disables the GRUB submenu, i.e., it enables all the available kernels to be listed on the main GRUB Menu itself instead of the “Advanced option for Arch Linux” option.
    `GRUB_DEFAULT=saved` # saves the last kernel used
    `GRUB_SAVEDEFAULT=true` # makes sure that grub uses the last selected kernel is used as default
    ```

- Re-Generate GRUB Configuration file

    `sudo grub-mkconfig -o /boot/grub/grub.cfg`

- Choose Kernel From GRUB During Boot

---

### **wireguard**

```bash
# Install wg
sudo pacman -S wireguard-tools
# Open wg
sudo wg-quic up wg0
# Close wg
sudo wg-quic down wg0
```

**wg0.conf**

`/etc/wireguard/wg0.conf`

```bash
[Interface]
PrivateKey = xxxxxxxxxxxxxxxxxxx
Address = 172.16.0.2/32
Address = 2606:4700:110:8fe8:bc78:d25a:896a:9696/128
DNS = 1.1.1.1
MTU = 1280
[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = 0.0.0.0/0
AllowedIPs = ::/0
Endpoint = xxx.xxx.xxx.xxx
```

**==配置步骤如下：==**

```bash
# Error: /usr/bin/wg-quick: line 32: resolvconf: command not found
sudo pacman -S openresolv 
```
1. WARP 密钥获取：Telegram 中 **Warp+ Bot** 获取

![image-20230722173251745](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221732768.png)

2. 配置文件生成：<https://replit.com/@misaka-blog/wgcf-profile-generator?v=1>

![image-20230722173030527](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221730585.png)

3. 优选IP **==warp-yxip==**

![image-20230722173556110](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221735152.png)
