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
