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

      ## 创建新项目、托管本地项目、克隆已有项目

