

[TOC]

---

# git

## 配置git

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


## git 常用命令

- `git status`
- `git clone`
- `git pull`
- `git push`
- `git commit -m 'commits'` or `git commit -m 'commits' xxx.fileType`
- `git add .` or `git xxx.fileType`
- `git reflog`
- 

# 服务器相关

## scp 文件上传、下载

- 上传 `scp .\cifar-10-python.tar.gz lutianen@10.170.46.236:/home/lutianen/`
- 下载  `scp root@100.100.100.100:/var/tmp/a.txt /var`

