# Arch Linux Manu

## Install Arch

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
    pacstrap /mnt bash base base-devel linux linux-headers linux-firmware neovim xsel

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
    sudo pacman -S plasma xorg nvidia dolphin konsole fish noto-fonts-cjk noto-fonts-emoji
    sudo systemctl enable sddm
    
    # reboot
    exit
    swapoff /mnt/swapfile
    umount -R /mnt
    reboot
    ```

---

## 内核更换

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

## Software

### Check NetworkManager

```bash
ping baidu.com
systemctl enable NetworkManager
```

---

### pacman 镜像修改

```bash
sudo vim /etc/pacman.conf
# ----------------
# Misc options
Color
ParallelDownloads = 5
# ---
[multilib]
Include = /etc/pacman.d/mirrorlist
#---
键入：
[archlinuxcn]
Server = https://mirrors.utsc.edu.cn/archlinuxcn/$arch
#---

sudo pacman -Syyu
sudo pacman -S archlinuxcn-keyring
```

---

### 常见通用软件

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

#### [输入法](https://wiki.archlinuxcn.org/wiki/Fcitx5)

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

yay -Sy neofetch google-chrome obs-studio baidunetdisk nutstore-experimental xunlei-bin telegram-desktop gitkraken visual-studio-code-bin typora-free redis net-tools pot-translation translate-shell okular snipaste gwenview kcalc wemeet-bin vlc wget ark shotcut inkscape ninja gnu-netcat tcpdump cmake clang tree python-pip caj2pdf-qt ttf-hack-nerd transmission-gtk gpick speedcrunch drawio-desktop crow-translate

yay -S electronic-wechat-uos-bin linuxqq lx-music-desktop
```

- **gpick**: 可以从桌面任何地方取色，并且它还提供一些其它的高级特性
- **SpeedCrunch**: 一个漂亮，开源，高精度的科学计算器
- **Snipaste**: 截图工具，如不可用可选用`spectacle`
- **drawio-desktop**: [Security-first diagramming for teams](https://github.com/jgraph/drawio-desktop)
- **crow-translate**：[翻译工具](https://github.com/crow-translate/crow-translate)

---

#### office

```bash
yay -S wps-office wps-office-mui-zh-cn ttf-wps-fonts
```

##### Problems

- **wps-office大部分字体粗体出现过粗无法正常显示问题**

    > 问题: freetype2更新至2.13.0以上版本后出现的问题。导致wps-office 文档编辑文字大部分字体设置粗体出现过粗无法正常显示。
    >
    > 解决方案：[freetype2 降级至 2.13.0]( https://bbs.archlinux.org/viewtopic.php?id=288562 )
    >
    >   1. Download[freetype2.13.0](https://pan.baidu.com/s/15AIkxKqvTwy9Q-DS16QQIQ?pwd=ft13)
    >   2. 降级 `sudo pacman -U freetype2-2.13.0-1-x86_64.pkg.tar.zst`
    >   3. 修改 `/etc/pacman.conf` -> `IgnorePkg = freetype2`，排除掉这个包（不让它更新） `freetype2: ignoring package upgrade (2.13.0-1 => 2.13.2-1)`
    >
    > env LD_LIBRARY_PATH=/usr/local/freetype2-2.13.0-1-x86_64/usr/lib
    >
    > `update-desktop-database ~/.local/share/applications

- `wpspdf 无法打开 PDF 文件`

    > wpspdf 依赖于 libtiff5.so.5 以支撑其 PDF 功能。而系统更新后，Arch Linux 提供的是 libtiff.so.6 或更新版本，导致其无法正常工作。
    >
    > 解决方案：安装 [libtiff5](https://aur.archlinux.org/packages/libtiff5/)

- `WPS 无法输入中文`

    > [解决方案](https://wiki.archlinuxcn.org/wiki/WPS_Office#Fcitx5_%E6%97%A0%E6%B3%95%E8%BE%93%E5%85%A5%E4%B8%AD%E6%96%87)
    >
    > `wpp`
    >
    > `wpspdf`
    >
    > `wpp`
    >
    > `et`

- `lx-music 数据同步失败`

    > 1. **确保PC端的同步服务已启用成功**: 若连接码、同步服务地址没有内容，则证明服务启动失败，此时看启用同步功能复选框后面的错误信息自行解决
    > 2. 在手机浏览器地址栏输入<http://x.x.x.x:5963/hello后回车，若此地址可以打开并显示> Hello~::^-^::~v4~，则证明移动端与PC端网络已互通，
    > 3. 若移动端无法打开第2步的地址，则在PC端的浏览器地址栏输入并打开该地址，若可以打开，则可能性如下：
    >    - LX Music PC端被**电脑防火墙**拦截
    >    - **PC端与移动端不在同一个网络下**，
    >    - 路由器开启了AP隔离（一般在公共网络下会出现这种情况）
    > 4. 要验证双方是否在同一个网络或是否开启AP隔离，可以在电脑打开cmd使用ping命令ping移动端显示的ip地址，若可以通则说明网络正常

---

### Git

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

#### Git 常用命令

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

#### 常见问题

1. `fatal: unable to access 'https://github.com/xxxxxxx.git/': Failed to connect to github.com port 443: Timed out`

   问题：代理出问题

   解决方案：

   ```bash
   git config --global --unset http.proxy
   git config --global --unset https.proxy
   ```

2. `fatal: unable to access 'https://github.com/xxxx.git/': gnutls_handshake() failed: The TLS connection was non-properly terminated.`

   解决方案：

   ```bash
   git config --global http.sslVerify false
   ```

---

### Golang

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

---

### MySQL

很多linux发行版都放弃了对mysql的支持（原因自行百度）转而支持mariadb（mysql的另一个分支），Archlinux就是其中之一，mariadb具有和mysql一模一样的操作命令，所以完全不用考虑迁移兼容的问题

- 安装mariadb: `sudo pacman -Sy mariadb`

- 配置mariadb命令，创建数据库都在/var/lib/mysql/目录下面: `sudo mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql`

- 开启mariadb 服务: `systemctl start mariadb`

- 初始化密码，期间有让你设置密码的选项，设置你自己的密码就行了，然后根据自己理解y/n就可，因为很多后面可以再修改: `sudo /usr/bin/mysql_secure_installation`

- 登录mariadb 和mysql命令是一样的: `mysql -u root -p`

- 设置开机自启动服务: `systemctl enable mariadb #开机自启动`

---

### you-get

命令行程序，提供便利的方式来下载网络上的媒体信息。

```bash
yay -S you-get
```

- 下载流行网站之音视频，例如YouTube, Youku, Niconico,以及更多
- 于您心仪的媒体播放器中观看在线视频，脱离浏览器与广告
- 下载您喜欢的网页上的图片
- 下载任何非HTML内容，例如二进制文件

---

### 挂载其他硬盘分区

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

![Present Windows](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202309141103383.png)

---

### `scp` 文件上传、下载

- 上传 `scp .\cifar-10-python.tar.gz lutianen@10.xxx.xxx.xxx:/home/lutianen/`
- 下载  `scp root@10.xxx.xxx.xxx:/var/tmp/a.txt /var`

---

### picgo

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

1. **安装 picgo**: `yay -S picgo`

2. **picgo 配置 github**

    - github 获取 **token**: 见 **picgo-core** 配置

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

sudo pacman -S virtualbox-guest-utils
sudo systemctl enable vboxservice.service
```

---

### CUDA & cuDNN

```bash
yay -S cuda-11.7 cudnn8-cuda11.0
```

Arch Linux 会将 CUDA 相关档案安装至 `/opt/cuda`，有需要的话可以将 CUDA 的 `PATH` 加到 `~/bashrc`，此路径永远指向最新版CUDA

```bash
# cuda # fish
set PATH /opt/cuda-11.7/bin $PATH
set LD_LIBRARY_PATH /opt/cuda-11.7/lib64/ $PATH

pip install torch==1.13.1+cu117 torchvision==0.14.1+cu117 torchaudio==0.13.1 --extra-index-url https://download.pytorch.org/whl/cu117
```

---

### wireguard

```bash
# Install wg
sudo pacman -S wireguard-tools
# Open wg
sudo wg-quick up wg0
# Close wg
sudo wg-quick down wg0
```

#### wg0.conf

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

#### 配置步骤如下

```bash
# Error: /usr/bin/wg-quick: line 32: resolvconf: command not found
sudo pacman -S openresolv
```

1. WARP 密钥获取：Telegram 中 **Warp+ Bot** 获取

![image-20230722173251745](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221732768.png)

2.配置文件生成：<https://replit.com/@misaka-blog/wgcf-profile-generator?v=1>

<https://replit.com/@tianenxd/wgcf-profile-generator，需要登陆>

![image-20230722173030527](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221730585.png)

3.优选IP **==warp-yxip==**

```bash
wget -N https://gitlab.com/Misaka-blog/warp-script/-/raw/main/files/warp-yxip/warp-yxip.sh && bash warp-yxip.sh
```

![image-20230722173556110](https://raw.githubusercontent.com/lutianen/PicBed/master/202307221735152.png)

---

### Clash Verge 解决DNS泄露问题

DNS 泄露其实并没有一个明确的定义，也不存在一个官方解释。

大概就是说你访问YouTube等黑名单网站的时候，使用中国大陆的DNS服务器进行了解析，这可能导致隐私问题的。

如果在 [DNS Leak Test](https://browserleaks.com/dns) 、[ipleak](https://ipleak.net/)这种网站的列表中看到了中国国旗，就要意识到可能发生了DNS泄露。
虽然没有人知道具体的探测机制是什么，但很可能是从网络层面获取的。在一般的家庭网络拓扑中，wireshark可以看到什么内容，运营商就能看见什么内容，所以你使用114.114.114.114、223.5.5.5这样的DNS解析去访问了什么网站是很清晰的。

**Clash开启TUN模式，关闭系统代理去使用**：与普通的系统代理模式区别在于，TUN模式下Clash会创建一张虚拟网卡，从网络层面接管所有的网络流量。

#### Step 1: 开启TUN模式

#### Step 2: 使用稳定的DNS

DNS这部分有人会教使用运营商的DNS，**运营商的DNS只适合小白用户，因为他可能连反诈**，所以建议使用国内大厂的。

1. 关闭浏览器的QUIC, 中国大陆的isp是限速udp的, 所以导致QUIC这个优秀的协议, 到了中国大陆的网络下成了个负面增益效果。

    `about://flags/#enable-quic` 设置为`Disabled` (点下方弹出的重启浏览器生效)

    <img src="../../../.config/Typora/typora-user-images/image-20240309001559678.png" alt="image-20240309001559678" style="zoom:50%;" />

2. 关闭浏览器中的“安全DNS”

    `chrome://settings/security`

    <img src="https://raw.githubusercontent.com/lutianen/PicBed/master/image-20240309001749185.png" alt="image-20240309001749185" style="zoom:50%;" />

3. 在Clash Verge的【Profiles】中，点右上角的"NEW" -> Type选择"Script" -> Name随意填写(例如，"修改DNS")

4. 右击新建的文件，然后"Edit File"，输入以下内容后启用：

    ```JavaScript
    function main(content) {
    const isObject = (value) => {
        return value !== null && typeof value === 'object'
    }

    const mergeConfig = (existingConfig, newConfig) => {
        if (!isObject(existingConfig)) {
        existingConfig = {}
        }
        if (!isObject(newConfig)) {
        return existingConfig
        }
        return { ...existingConfig, ...newConfig }
    }

    const cnDnsList = [
        'tls://223.5.5.5',
        'tls://1.12.12.12',
    ]
    const trustDnsList = [
        'https://doh.apad.pro/dns-query',
        'https://dns.cooluc.com/dns-query',
        'https://1.0.0.1/dns-query',
    ]
    const notionDns = 'tls://dns.jerryw.cn'
    const notionUrls = [
        'http-inputs-notion.splunkcloud.com',
        '+.notion-static.com',
        '+.notion.com',
        '+.notion.new',
        '+.notion.site',
        '+.notion.so',
    ]
    const combinedUrls = notionUrls.join(',');
    const dnsOptions = {
        'enable': true,
        'default-nameserver': cnDnsList, // 用于解析DNS服务器 的域名, 必须为IP, 可为加密DNS
        'nameserver-policy': {
        [combinedUrls]: notionDns,
        'geosite:geolocation-!cn': trustDnsList,
        },
        'nameserver': trustDnsList, // 默认的域名解析服务器, 如不配置fallback/proxy-server-nameserver, 则所有域名都由nameserver解析
    }

    // GitHub加速前缀
    const githubPrefix = 'https://ghproxy.lainbo.com/'

    // GEO数据GitHub资源原始下载地址
    const rawGeoxURLs = {
        geoip: 'https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip-lite.dat',
        geosite: 'https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat',
        mmdb: 'https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country-lite.mmdb',
    }

    // 生成带有加速前缀的GEO数据资源对象
    const accelURLs = Object.fromEntries(
        Object.entries(rawGeoxURLs).map(([key, githubUrl]) => [key, `${githubPrefix}${githubUrl}`]),
    )

    const otherOptions = {
        'unified-delay': true,
        'tcp-concurrent': true,
        'profile': {
        'store-selected': true,
        'store-fake-ip': true,
        },
        'sniffer': {
        enable: true,
        sniff: {
            TLS: {
            ports: [443, 8443],
            },
            HTTP: {
            'ports': [80, '8080-8880'],
            'override-destination': true,
            },
        },
        },
        'geodata-mode': true,
        'geox-url': accelURLs,
    }
    content.dns = mergeConfig(content.dns, dnsOptions)
    return { ...content, ...otherOptions }
    }
    ```

5. 设置完成后，验证DNS解析结果是否都是来自国外的Cloudflare和Google的DNS, 这时节点服务器不管拿到了你传过去的真ip还是假ip地址, 他都会再去请求一次Cloudflare/Google的DNS服务, 确保解析的正确性。
    重要的是**没有中国大陆的DNS服务器**了，如果还是有，那你应该往当前设备的更上层寻找问题所在，比如路由器的设置等。

#### Clash Verge 解决 GEOIP，CN问题

目前市面上绝大多数的代理工具都依赖于 GeoIP2 数据库判断地址所属地。它们的规则结尾部分一般都会有一条类似 `GEOIP, CN`，用来查询目的 IP 地址是否属于中国大陆，从而判断是否直连。

这些代理工具通常使用的 GeoIP2 数据库是来自于 MaxMind 的 [GeoLite2](https://dev.maxmind.com/geoip/geoip2/geolite2/) 免费数据库。这个数据库目前存在一下几个问题：

- 获取不便：从 2019 年 12 月 30 日起，必须注册后才能下载

- 数据量大：数据库庞大，包含全球的 IP 地址段，约 10 MB

- 准确度低：对中国大陆的 IP 地址判定不准，如：香港阿里云的 IP 被判定为新加坡、中国大陆等。

庞大的数据量对于大多数中国大陆的用户来说是没有意义的，因为只仅需要去判断 IP 的地理位置是否属于中国大陆境内，其他国家的 IP 一律代理/直连。过多的数据量会增加载入时间，降低查询效率。

我们在之前创建的Script中已经包含了下载更精简合适中国大陆的IP数据库链接, 现在只需要手动操作下载和替换即可:

1. **Update GeoData**: Clash Verge Rev的`设置`菜单中点击`Update GeoData`
2. **验证下载**: 打开Clash Verge托盘中的`APP Dir`，找到`geoip.dat`文件，验证其大小是否为**几百KB**
3. **重启Clash Verge**：确保数据库被正确应用

## System optimization

### SSD 优化

**TRIM**, 会帮助清理SSD中的块，从而延长SSD的使用寿命

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

---

### SWAP 设置

<https://wiki.archlinux.org/title/Swap#Swappiness>

- 查看 swap 使用率，一般是 60 ，意思是 60% 的概率将内存整理到 swap: `cat /proc/sys/vm/swappiness`

- 修改 swap 使用策略为 10%，即 10% 的概率将内存整理到 swap: `sudo sysctl -w vm.swappiness=10`

- 修改配置文件：`sudo vim /etc/sysctl.d/99-swappiness.conf` 在文件末尾加上下面这行内容：`vm.swappiness=10`

- 重启后可查看 swappiness 的值

    ![image-20230723115427188](https://raw.githubusercontent.com/lutianen/PicBed/master/202307231154321.png)

---

### Systemd journal size limit

控制日志最大可使用多少磁盘空间，修改`/etc/systemd/journald.conf` 中的`SystemMaxUse`参数 `SystemMaxUse=50M`

---

## Games

- sdl-ball: `yay -S sdl-ball`

## Problems

1. `clear` command - `terminals database is inaccessible`

   解决方案：[Path for Anaconda3 is set in `.bashrc`. It is interfering with the `clear` command. Removing Anaconda path from path solved the issue.](https://github.com/ContinuumIO/anaconda-issues/issues/331)

   > ```bash
   > ~  echo $CONDA_PREFIX                                       (base) 
   > /opt/miniconda
   > sudo mv $CONDA_PREFIX/bin/clear $CONDA_PREFIX/bin/clear_old
   > ```

2. `tput: unknown terminal "xterm-256color"`

   解决方案：`setenv TERMINFO /usr/lib/terminfo`wps

3. **更新内核后，双屏显示时，某一个屏幕黑屏，但鼠标能够移动过去并显示，另一屏幕正常**

   解决方案：`xrandr --output HDMI-1-0 --right-of eDP1 --auto`

   - 命令解释：配置 `HDMI-1-0` 输出，使其位于 `eDP1` 输出的右侧，并自动选择最佳的分辨率和刷新率设置

   > ```bash
   > $ xrandr --listmonitors
   > Monitors: 2
   > 0: +*eDP1 2560/360x1440/200+0+0  eDP1
   > 1: +HDMI-1-0 1920/479x1080/260+2560+0  HDMI-1-0
   > $ xrandr --output HDMI-1-0 --right-of eDP1 --auto
   > ```
