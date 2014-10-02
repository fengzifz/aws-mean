# 在 AWS 上部署 MEAN 开发环境

### 前言

 - 该文档不会说如何在 AWS 上面创建 EC2 实例，如有需要，请自行参考 AWS 的文档
 - 该文档所说的内容，是针对 **Amazon Linux 64bit**，不排除其他 AMI 会不适用

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