# 在 AWS 上部署 MEAN 开发环境

### 前言

 - 该文档不会说如何在 AWS 上面创建 EC2 实例，如有需要，请自行参考 AWS 的文档
 - 该文档所说的内容，是针对 **Amazon Linux 64bit**

------

### 安装 MongoDB

#### 为 MongoDB 创建 Amazon EBS volumes

EBS 用来永久保存数据，我们创建一个 EBS 后，可以随便挂在到任意的 Instance 上。

- 登录 [AWS](http://aws.amazon.com/)，进入 EC2 管理界面
- 点击 **ELASTIC BLOCK STORE > Volumes > Create Volume** 创建（**注意：Amazon 会按照你所选择的 EBS 容量进行收费，建议一开始不要选择太多，以后可以按需扩容。[EBS 收费](http://aws.amazon.com/cn/ebs/pricing/)**）
- 右键刚刚创建的 volume，点击 **"Attach Volume"**，把它挂载到你的 Instance 上


#### 在EBS  volume 上创建文件系统

在上一个步骤中，创建了 EBS 并挂载到 Instance 后，你的这个 Instance 会创建一个`/dev/sdf`，然后我们在 terminal 里面执行：

```bash
sudo mkfs -t ext4 /dev/sdf
```

#### 挂载驱动器
```bash
sudo mkdir -p /data/db
sudo chown `id -u` /data/db
```

#### 启动时列出 volume 
```bash
sudo echo `/dev/sdf /data/db auto noatime,noexec,nodiratime 0 0` >> /etc/fstab
```

**注意：** 如果在这里遇到权限问题，可以切换到 root 用户在执行
```bash
# root user
sudo -s
```

#### 挂载 volume
```bash
sudo mount -a /dev/sdf /data/db
```

#### 下载 MongoDB
```bash
curl http://fastdl.mongodb.org/linux/mongodb-linux-x86_64-2.0.2.tgz > m.tgz
tar xzf m.tgz
```

#### 使用 package manager 安装
```bash
sudo vi /etc/yum.repos.d/10gen.repo

# Then edit the file to contain:
name=10gen Repository
baseurl=http://download-distro.mongodb.org/repo/redhat/os/x86_64
gpgcheck=0
```

#### 使用 yum 安装
```bash
sudo yum install mongo-10gen
```

#### 配置 MongoDB
```bash
sudo vi /etc/mongodb.conf

# Edit the file to contain:
oplogSize=8000
journal=true
dbpath=/data/db
logpath=/data/db/logs
```

#### 启动 MongoDB
```bash
./mongodb-xxxxxx/bin/mongod
```

#### 自动启动 MongoDB
```bash
chkconfig mongod on
```

------

### MongoDB - error:13 Permission deny
安装好 MongoDB 后，使用命令 `mongod` 启动 Mongo 时，可能会遇到 permission deny 的问题：

```bash
exception in initAndListen: 10309 Unable to create/open lock file: /data/db/mongod.lock errno:13 Permission denied Is a mongod instance already running?, terminating
```

这是因为当前的用户没有权限访问 `/data/db` 目录

```bash
# To fix it
chown <username> /data/db
```
------

### 安装 NodeJs

网上绝大部分教程都是使用 `apt-get` 来安装 node，但 Amazon Linux 64bit 里面是没有 apt 组件的，因此，我们使用源码直接编译安装。

#### 安装 gcc-c++
```bash
yum install gcc-c++
```

#### 下载 NodeJs
```bash
sudo wget http://nodejs.org/dist/v0.10.32/node-v0.10.32.tar.gz
```

#### 解压文件，并检查配置
```bash
cd ~/node-v0.10.32.tar.gz
tar xvf node-v0.10.32.tar.gz

cd node-v0.10.32

# 如果不先安装 gcc-c++，都这里会报错的
./configure
```

#### 安装 NodeJs
安装可能需要等待的时间比较长
```bash
make install
```

#### 检查安装是否成功
```bash
node --version
```

如果能够成功返回 node 的版本号，说明已经安装成功

------

### -bash: node: command not found

如果出现 `-bash: node: command not found` 错误，先检查 node 是否已经真的安装了：

```bash
ls -l /usr/local/bin | grep node

# Bash display below message is mean that node have existed
>> -rwxrwxr-x 1 root root 11515348 10月  1 03:48 node
>> lrwxrwxrwx 1 root root       38 10月  1 03:52 npm -> ../lib/node_modules/npm/bin/npm-cli.js


ls -l /usr/local/bin/node

# Bash desplay below message
>> -rwxrwxr-x 1 root root 11515348 10月  1 03:48 /usr/local/bin/node
```

如果出现上面的结果，那么 node 已经安装了，那么可能就是编译路径未配置，配置编译路径，在 `/etc/profile` 文件里面，添加两行：

```bash
export PATH="$HOME/local/node/bin:$PATH"
export NODE_PATH="$HOME/local/node:$HOME/local/node/lib/node_modules"
```

然后重启系统：

```bash
reboot -f
```

------

### -bash: npm: command not found

安装好 Node 后，可以使用 npm 来安装 AngularJS 和 Express，但使用 npm 的时候，可能会出现错误：`-bash: npm: command not found`

在 terminal 里面输入以下命令解决：

```bash
sudo ln -s /usr/local/bin/node /usr/bin/node
sudo ln -s /usr/local/lib/node /usr/lib/node
sudo ln -s /usr/local/bin/npm /usr/bin/npm
sudo ln -s /usr/local/bin/node-waf /usr/bin/node-waf
```

------

### 安装 ExpressJS 和 AngularJS

```bash
npm install express
npm install angular
```

------

### 开放 AWS EC2 Instance 的 80 端口

- Go to the Security Group settings in the left hand navigation
- Find the Security Group that your instance is apart of
- Click on Inbound Rules
- Use the drop down and add HTTP (port 80)
- Click Apply and enjoy