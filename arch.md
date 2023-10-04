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
    pacman -S networkmanager bluez bluez-utils pulseaudio-bluetooth alsa-utils pulseaudio pulseaudio-alsa sof-firmware
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
    # 取消问候语
    set -U fish_greeting ""

    sudo vim /etc/systemd/system/clash.service
    sudo systemctl daemon-reload 
    sudo systemctl enable clash 
    sudo systemctl start clash 
    sudo systemctl status clash

    sudo pacman -S obs-studio
    ```

    [Inputs](https://wiki.archlinuxcn.org/wiki/Fcitx5)

    ```bash
    sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons fcitx5-material-color fcitx5-pinyin-moegirl fcitx5-pinyin-zhwiki

    # sudo vim /etc/environment
    GTK_IM_MODULE=fcitx
    QT_IM_MODULE=fcitx
    XMODIFIERS=\@im=fcitx
    # 为了让一些使用特定版本 SDL2 库的游戏能正常使用输入法
    SDL_IM_MODULE=fcitx
    ```

    ```bash
    yay -S clash-for-windows-bin 

    yay -Sy neofetch google-chrome obs-studio baidunetdisk nutstore-experimental xunlei-bin telegram-desktop libreoffice-still libreoffice-still-zh-cn gitkraken visual-studio-code-bin typora-free redis net-tools pot-translation translate-shell okular spectacle gwenview kcalc wemeet-bin vlc wget ark shotcut inkscape ninja gnu-netcat tcpdump cmake clang
    
    yay -S electronic-wechat-uos-bin linuxqq lx-music-desktop
    ```

4. you-get

    命令行程序，提供便利的方式来下载网络上的媒体信息。

    ```bash
    yay -S you-get
    ```

    - 下载流行网站之音视频，例如YouTube, Youku, Niconico,以及更多
    - 于您心仪的媒体播放器中观看在线视频，脱离浏览器与广告
    - 下载您喜欢的网页上的图片
    - 下载任何非HTML内容，例如二进制文件

5. Golang

    ```bash
    # Download and install go
    sudo pacman -S go

    # Set environment variable in `.config/fish/config.sh` or `/etc/profile` or `~/.profile`
    GOROOT /usr/lib/go
    GOPATH /home/tianen/goProj
    GOBIN /home/tianen/goProj/bin
    PATH $GOPATH/bin $GOROOT/bin $GOBIN $PATH
    ```

    - **`GOROOT`，设置 Golang 的安装位置**
    - **`GOBIN`，执行 `go install` 后生成可执行文件的目录**
    - **`GOPATH`，工作目录，一般设置到用户目录下**

        ```bash
        # Go 工作目录结构
        ├── bin  # 存放 `go install` 命令生成的可执行文件，且可把 `$GOBIN` 路径加入到 `PATH` 环境变量中，这样就可以直接在终端中使用 go 开发生成的程序
        ├── pkg # 存放 go 编译生成的文件
        ├── readme.md
        └── src # 存放 go 源码，不同工程项目的代码以包名区分
        ```

6. 优化

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
   git config --global user.email tianen.xd@gmail.com
   
   # check
   git config --list
   ```

2. 生成密钥

   ```bash
   ssh-keygen.exe -t rsa -C 'tianen.xd@gmail.com'
   ```

   > 上述代码执行完成后，要求多次输入密码，**请不要输入密码**

3. github配置 SSH Keys

   1. 打开生成的 `Key` 文件 `/.ssh/id_rsa.pub`

   2. 复制全部内容，在 Key 中粘贴

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

### **挂载其他硬盘分区**

```bash
# Get UUID and TYPE
sudo blkid

# /dev/nvme1n1p3: LABEL="Document" BLOCK_SIZE="512" UUID="111915F1111915F1" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="666266ba-233b-11ed-95be-00e04c3656eb"

# Write UUID TYPE ...
sudo vim /etc/fstab

# <device> <dir> <type> <options> <dump> <fsck>
UUID=111915F1111915F1 /home/tianen/doc ntfs3 defaults 0 0
```

- `<device>` 描述要挂载的特定块设备或远程文件系统
- `<dir>` 描述挂载目录
- `<type>` 文件系统类型
- `<options>` 相关的挂载选项
- `<dump>` 会被 dump(8) 工具检查。该字段通常设置为 0, 以禁用检查
- `<fsck>` 设置引导时文件系统检查的顺序; 对于 root 设备该字段应该设置为 1。对于其它分区该字段应该设置为 2,或设置为 0 以禁用检查

**我使用 TYPE 为 `ntfs` 时导致启动失败，修改为 `ntfs3` 后成功挂载**

### Present Windows

<img src="https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202309141103383.png" alt="image-20230914110307330" style="zoom:67%;" />

### scp

**文件上传、下载**

- 上传 `scp .\cifar-10-python.tar.gz lutianen@10.170.46.236:/home/lutianen/`
- 下载  `scp root@100.100.100.100:/var/tmp/a.txt /var`

### picgo 配置

#### picgo-core 【Recommend】

1. Download and Install **PigGo-Core**

   ![image-20231004132814030](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/image-20231004132814030.png)

2. Get **token** with GitHub

   ![Token](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/Screenshot_20230912_221106.png)

3. Configure `config.json`

   **NOTE：When using `~/.picgo/config.json`, delete the comments to avoid unnecessary trouble（使用时，将注释删掉，以免产生不必要的麻烦）**

   ```json
   {
     "picBed": {
       "current": "github",
       "github": {
         "repo": "lutianen/PicBed", // 设定仓库名：上文在 GitHub 创建的仓库 `lutianen/PicBed`
         "branch": "master", // 设定分支名：`master`
         "token": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", // 设定 Token：上文生成的 toke
         "path": "", // 指定存储路径：为空的话会上传到根目录，也可以指定路径
         "customUrl": "" // 设定自定义域名：可以为空
       },
       "uploader": "github",
       "transformer": "path"
     },
     "picgoPlugins": {
       "picgo-plugin-github-plus": true
     }
   }
   ```

#### picgo app 【Not Recommend】

**安装 picgo**

```bash
yay -S picgo
```

**picgo 配置 github**

- github 获取 **token**

- PicGo 配置

  ![image-20230912222947951](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/image-20230912222947951.png)

  - 设定仓库名：上文在 GitHub 创建的仓库 `lutianen/PicBed`
  - 设定分支名：`master`
  - 设定 Token：上文生成的 token
  - 指定存储路径：为空的话会上传到跟目录，也可以指定路径
  - 设定自定义域名：可以为空，这里为了使用 CDN 加快图片的访问速度，按这样格式填写：[https://cdn.jsdelivr.net/gh/lutianen/PicBed/@master](https://cdn.jsdelivr.net/gh/lutianen/PicBed/@master)

---

### VirtualBox

```bash
sudo pacman -S virtualbox  # dkms

sudo pacman -S virtualbox-host-dkms
sudo dkms autoinstall
sudo modprobe vboxdrv
```

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

    ![image-20230723115427188](https://raw.githubusercontent.com/lutianen/PicBed/master/202307231154321.png)

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
sudo wg-quick up wg0
# Close wg
sudo wg-quick down wg0
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

![image-20230722173251745](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221732768.png)

2. 配置文件生成：<https://replit.com/@misaka-blog/wgcf-profile-generator?v=1>

![image-20230722173030527](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221730585.png)

3. 优选IP **==warp-yxip==**

![image-20230722173556110](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221735152.png)
