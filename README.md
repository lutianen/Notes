# Learning Notes

[TOC]

---

## git

### 配置git

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

### git 常用命令

- `git status`
- `git clone`
- `git pull`
- `git push`
- `git commit -m 'commits'` or `git commit -m 'commits' xxx.fileType`
- `git add .` or `git xxx.fileType`
- `git reflog`

---

### Git实现从本地添加项目到远程仓库

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
>      - *由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，
>      - *在以后的推送或者拉取时就可以简化命令*

## 服务器相关

### scp 文件上传、下载

- 上传 `scp .\cifar-10-python.tar.gz lutianen@10.170.46.236:/home/lutianen/`
- 下载  `scp root@100.100.100.100:/var/tmp/a.txt /var`


## Qt 5.15.3 源码编译

### Windows

1. msys2，将linux下的很多工具和库移植到了windows上，并提供一套类linux的shell环境和完善的编译工具链，方便开发者编译，安装，运行原生windows软件。通过使用msys2的Arch Linux包管理工具pacman，可以很方便自由地安装我们需要的库

#### System requirements 

> pen a command prompt and ensure that the following tools can be found in the path : `perl -v`、`gcc -v`、`python -V`、`ruby -v`
>
> - MinGW-builds gcc 4.9 or later
> - Perl version 5.12 or later
> - Python version 2.7 or later
> - Ruby version 1.9.3 or later

#### 下载地址

- MinGW --- https://sourceforge.net/projects/mingw-w64/files/mingw-w64/mingw-w64-release/

  > ```powershell
  > # 测试 C++ compiler supporting the C++11 standard
  > F:\qt-everywhere-src-5.15.2> mingw32-make -v
  > 
  > GNU Make 4.2.1
  > Built for i686-w64-mingw32
  > Copyright (C) 1988-2016 Free Software Foundation, Inc.
  > License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
  > This is free software: you are free to change and redistribute it.
  > There is NO WARRANTY, to the extent permitted by law.
  > ```

- Perl --- https://www.perl.org/get.html

  > ```powershell
  > # 测试
  > F:\qt-everywhere-src-5.15.2> perl -v
  > 
  > This is perl 5, version 32, subversion 1 (v5.32.1) built for MSWin32-x86-multi-thread-64int
  > 
  > Copyright 1987-2021, Larry Wall
  > 
  > Perl may be copied only under the terms of either the Artistic License or the
  > GNU General Public License, which may be found in the Perl 5 source kit.
  > 
  > Complete documentation for Perl, including FAQ lists, should be found on
  > this system using "man perl" or "perldoc perl".  If you have access to the
  > Internet, point your browser at http://www.perl.org/, the Perl Home Page
  > ```

- Ruby --- https://rubyinstaller.org/downloads/

  > ```powershell
  > # 测试
  > C:\Users\Tianen>ruby -v
  > ruby 3.1.1p18 (2022-02-18 revision 53f5fc4236) [i386-mingw32]
  > ```

#### Build

```powershell
# Step 1. 设置临时环境变量 F:\Strawberry\c\bin;
set path=F:\Python\Python38\;F:\Python\Python38\Scripts;C:\Windows;C:\Windows\system32;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH;F:\Strawberry\perl\site\bin;F:\Strawberry\perl\bin;F:\MinGW\bin;F:\Ruby31\bin;F:\LLVM\bin;F:\cmake32\bin;
set LANG=en
cmd /k

# Step 2. 
# -prefix 指定安装将会部署的位置，根据自己情况修改
# -debug-and-release 指示编译生成debug版和release版的Qt库
# -platform win32-g++ 指明编译平台是windows，并使用mingw编译器
# -opensource -confirm-license 是为了自动确认开源证书，免得到时暂停手动确认
# -nomake tests 不需要编译测试工程
# -skip qtwebengine 暂时先不编译webengine模块，因为太大了
# -qt-zlib -ssl -icu 指示检测这些库，并在需要时使用
# -opengl desktop 明确指示使用你windows上安装的opengl驱动来编译程序，但这样编译出的程序在别的电脑上运行时需要目标电脑上安装的opengl驱动能兼容你的程序
..\configure.bat -prefix F:\Qt\Qt5.15.3 -confirm-license -opensource -platform win32-g++ -debug -static -qt-sqlite -qt-zlib -qt-libpng -qt-libjpeg -opengl desktop -qt-pcre -qt-freetype -nomake tests -nomake examples -skip qtwebengine

# rm -rf configure.cache

./configure -prefix /usr/local/Qt/Qt5.15.3/ -confirm-license -opensource -mp -debug-and-release -static -qt-sqlite -qt-zlib -qt-libpng -qt-libjpeg -qt-freetype -debug -nomake tests -nomake examples -skip qtwebengine

# Step 3. make
mingw32-make -j16    # 开启十六线程编译，根据自己电脑实际核心数调整

# Step 4. make install 
mingw32-make install

# Step 5. 安装帮助文档
# 编译qch格式文档
mingw32-make docs
# 安装文档
mingw32-make install_doc
```



#### Problem

1. `Project ERROR: msvc-version.conf loaded but QMAKE_MSC_VER isn't set`

   > 动清空工程原来编译后生成的文件夹及文件，重新生成工程即可。
   >
   > 删除和项目有关的所有.qmake.stash以及构建目录(包含同级、上级、上上级,只要相关的.qmake.stash
   
2. 在添加Qt Versions时可能会报“qmlscene 未安装”，出现黄色感叹号。这时你只需将Qt官方动态编译的目录下的“qmlscene.exe”拷贝到自己编译的静态库的相应目录里面就ok了。

3. 您必须在 sources.list 中指定代码源(deb-src) URI

   > 原因是我们的文件/etc/apt/source.list里的deb-src都被注释掉了，而现在我们需要，找到问题了就好解决了，可以直接[vim](https://so.csdn.net/so/search?q=vim&spm=1001.2101.3001.7020)修改该文件把deb-src的注释去掉，也可以运行“软件和更新”修改，选中下图中的“源代码”

### qmake 使用

> - Usage: qmake [mode] [options] [files]
>
> - QMake has two modes, one mode for generating project files based on some heuristics, and the other for generating makefiles. Normally you shouldn't need to specify a mode, as makefile generation is the default mode for qmake, but you may use this to test qmake on an existing project.
>
> - 

```powershell
# 编写代码，命名为hello.cpp

# 创建.pro文件，将所有的文件编译成一个与平台无关的工程文件
qmake -project xxx.cpp -o xxx.pro

# 生成test.pro的Makefile
qmake -makefile test.pro

# make
make

# 执行
./xxx
```
