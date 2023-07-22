# Manjaro / ArchLinux's Manual

## Arch Linux 配置 PPPOE

1. 安装 ppp

    ```bash
    sudo pacman -S ppp
    ```

2. 使用 `ifconfig`查看 network interface

    找到网卡名称，一般可能是`eth0`，**然后把相应的网卡 trun down**

    ```bash
    ifconfig
    
    # Out
    ...
    eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            ether e0:70:ea:a8:da:c9  txqueuelen 1000  (Ethernet)
            RX packets 1804  bytes 574787 (561.3 KiB)
            RX errors 0  dropped 886  overruns 0  frame 0
            TX packets 1226  bytes 395773 (386.4 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ...
    
    # turn down eno1
    inconfig eno1 down
    ```

3. 配置 `PPPOE`相关配置信息

    ```bash
    sudo pppoe-setup
    
    # Result
    Ethernet Interface: eno1
    User name: 21011210016
    Activate-on-demand: No
    DNS addresses: 8.8.8.8
    Firewalling: NONE
    ```

4. 启动 `PPPPOE`

    ```bash
    sudo pppoe-start
    ```

5. 关闭 `PPPOE`

    ```bash
    sudo pppoe-stop
    ```

6. 查看 `PPPOE` 状态

    ```bash
    pppoe-status
    ```

---

## wireguard

```bash
# Install wg
sudo pacman -S wireguard-tools
# Open wg
sudo wg-quic up wg0
# Close wg
sudo wg-quic down wg0
```

**==wg0.conf==**

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

1. WARP 密钥获取：Telegram 中 **Warp+ Bot** 获取

![image-20230722173251745](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221732768.png)

2. 配置文件生成：https://replit.com/@misaka-blog/wgcf-profile-generator?v=1

![image-20230722173030527](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221730585.png)

3. 优选IP **==warp-yxip==**

![image-20230722173556110](https://raw.githubusercontent.com/lutianen/PicBed/main/202307221735152.png)

## 换源

```sh
sudo pacman-mirrors -i -c China -m rank
pacman -S archlinuxcn-keyring

# /etc/pacman.conf 添加
# [archlinuxcn]
# SigLevel = Optional TrustedOnly
# Server = https://mirrors.bfsu.edu.cn/archlinuxcn/$arch
```

## 常用软件

```bash
sudo pacman -S yay

# 清理日志
journalctl --vacuum-size=50M

sudo vim /etc/systemd/system/clash.service
sudo systemctl daemon-reload 
sudo systemctl enable clash 
sudo systemctl start clash 
sudo systemctl status clash
```

```bash
pacman -S package_name        # 安装软件
pacman -S extra/package_name  # 安装不同仓库中的版本
pacman -Syu                   # 升级整个系统，y是更新数据库，yy是强制更新，u是升级软件
pacman -Ss string             # 在包数据库中查询软件
pacman -Si package_name       # 显示软件的详细信息
pacman -Sc                    # 清除软件缓存，即/var/cache/pacman/pkg目录下的文件

pacman -R package_name        # 删除单个软件
pacman -Rs package_name       # 删除指定软件及其没有被其他已安装软件使用的依赖关系

pacman -Qs string             # 查询已安装的软件包
pacman -Qi package_name       # 查询本地安装包的详细信息
pacman -Ql package_name       # 获取已安装软件所包含的文件的列表

pacman -U package.tar.zx      # 从本地文件安装
pactree package_name          # 显示软件的依赖树

--noconfirm 无需确认
--noprogressbar 无进度静默
```

```bash
yay -Ss xxx #以xxx位关键字查找包
yay -S xxx  #安装xxx包（名字必须完全匹配）
yay -Sc     #清理软件包缓存
yay -Syyu   #全面更新

yay -Qs xxx  #以xxx为关键字查找已安装的包
yay -Qi xxx #查看软件包详细信息
yay -Ql xxx #查看所有某软件包有关的文件
yay -Qo [file] # 查看某个文件由哪个软件包拥有

yay -R xxx  #卸载包
yay -Rns xxx #卸载包以及无用的依赖

yay -U xxx  #xxx为本地包的地址，从离线包安装
```

```bash
sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons fcitx5-material-color
yay -Sy clash clash-for-windows-bin latte-dock visual-studio-code-bin lx-music linuxqq google-chrome baidunetdisk
```

**OBS**

```bash
sudo pacman -S obs-studio
```

- OBS 硬件编码注意点：需要选择最新版的显卡驱动，否则硬件编码会失败（只能用软编码）

    ![显卡选择最新版](https://raw.githubusercontent.com/lutianen/PicBed/main/202307131642756.png)

## python 脚本编写

应用python编写shell脚本经常要用到os,shutil,glob(正则表达式的文件名),tempfile(临时文件),pwd(操作/etc/passwd文件),grp(操作/etc/group文件),commands(取得一个命令的输出)

1. 解析命令行参数

    ```python
    #!/usr/bin/python3
    # -*- coding: utf-8 -*-
    
    # 
    # dest 参数指定解析结果被指派给属性的名字
    # metavar 参数被用来生成帮助信息
    # action 参数指定跟属性对应的处理逻辑，通常为 store，被用来存储 参数值 或将 多个参数值 收集到列表中
    # nargs 参数收集所有剩余的命令行参数，将它们放到一个列表中
    # 
    
    import argparse
    
    # The description
    parser = argparse.ArgumentParser(description="This is a test program.")
    #
    parser.add_argument(dest="", metavar="", nargs="*")
    
    # Add arguments
    parser.add_argument("-p", "--pat", metavar="pattern", required=True,
                        dest="patterns", action="append", help="test pattern to search for")
    parser.add_argument("-v", dest="verbose",
                        action="store_true", help="verbose mode")
    parser.add_argument("-o", dest="outfile", action="store", help="output file")
    parser.add_argument("--speed", dest="speed", action="store", choices={"slow", "fast"}, default="slow",
                        help="search speed")
    args = parser.parse_args()
    
    # Output results
    print(args.filenames)
    print(args.patterns)
    print(args.verbose)
    print(args.outfile)
    print(args.speed)
    ```

2. 密码处理

    **运行时需要一个密码。此脚本是交互式的，因此不能将密码在脚本中硬编码，而是需要弹出一个密码输入提示，让用户自己输入**

    ==python 中的 `getpass` 模块，可以让你很轻松的弹出密码输入提示，并且不会在用户终端回显密码==

    ```python
    #!/usr/bin/python3
    # -*- coding: utf-8 -*-
    
    import getpass
    
    # Get username
    user = getpass.getuser()
    print("Current user: ", user)
    # Get password with hidden input
    password = getpass.getpass()
    
    def svc_login(user, password):
        return user == password
    if svc_login(user, password):
        print('Yay!')
    else:
        print('Boo!')
    ```

    

