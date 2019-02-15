# 一 容器环境搭建

## 1.1 主机配置

### 1.1.1 配置要求

硬件需求根据Rancher部署的规模进行扩展。根据需求配置每个节点。

部署大小 | 集群(个) | 节点(个) | CPU | 内存
---|---|---|---|---
小 | 不超过5 | 最多50 | 4 | 16GB
中 | 不超过100 | 最多500 | 8 | 32GB
大 | 超过100 | 超过500 | 联系Rancher

### 1.1.2 操作系统选择

* Ubuntu 16.04(64位)   √
* Centos/RedHat Linux 7.5+(64位)
* RancherOS 1.3.0+(64位)
* Windows Server 1803(64位)

这里本人使用的是Ubuntu 16.04(64位)

一些常用的系统组件：
```
# 先更新apt-get源
sudo apt-get update
# 安装vim
sudo apt-get install vim
# 安装ssh
sudo apt-get install openssh-server
```

设置静态IP:
修改/etc/network/interfaces文件。注意：是interfaces，有s。
```
sudo vim /etc/network/interfaces
```
添加
```
address 192.168.4.246
netmask 255.255.255.0
gateway 192.168.4.1
dns-nameservers 8.8.8.8
```

### 1.1.3 Docker版本选择
支持的Docker版本

* 1.12.6
* 1.13.1
* 17.03.2
* 17.06 (for Windows)

```
注意
1. Ubuntu、Centos操作系统有Desktop和Server版本，选择请安装server版本，别自己坑自己！
2. 如果你正在使用RancherOS，请确保切换到受支持的Docker版本:
sudo ros engine switch docker-17.03.2-ce
```

### 1.1.4 主机名配置
因为K8S的规定，主机名只支持包含 - 和 .(中横线和点)两种特殊符号，并且主机名不能出现重复。

ubuntu 修改主机名
```
1. 修改hostname文件
sudo vim /etc/hostname
2. 这个文件中的内容是用来显示主机名字的，修改这个文件后，如果立刻重启，我们会看到终端中@后面的主机名将变为bbb
3. 修改hosts文件,修改127.0.0.1对应的主机名
4. 改完主机名字，我们需要重启计算机，否则命令执行会有些慢。
```

### 1.1.5 配置host
配置每台主机的hosts(/etc/hosts),添加host_ip $hostname到/etc/hosts文件中。
```
192.168.28.128  master
192.168.28.129  node02
```

### 1.1.6 关闭防火墙(或者放行端口)

1、Ubuntu
```
sudo ufw disable
```

### 1.1.7 配置主机时间、时区、系统语言
查看时区
```
date -R或者timedatectl
```
修改时区
```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
修改系统语言环境
```
sudo echo 'LANG="en_US.UTF-8"' >> /etc/profile;source /etc/profile
```
配置主机NTP时间同步

### 1.1.8 Kernel性能调优
```
cat >> /etc/sysctl.conf<<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
EOF
```
数值根据实际环境自行配置，最后执行sysctl -p保存配置。

## 1.2 Docker安装与配置
### Ubuntu
* 修改系统源
```
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list << EOF

deb http://mirrors.aliyun.com/ubuntu/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe

EOF
```

* Docker-ce安装
```
# 定义安装版本
export docker_version=17.03.2
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common bash-completion
# step 2: 安装GPG证书
sudo curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
version=$(apt-cache madison docker-ce|grep ${docker_version}|awk '{print $3}')
# --allow-downgrades 允许降级安装
sudo apt-get -y install docker-ce=${version} --allow-downgrades
# 设置开机启动
sudo systemctl enable docker
```

* 验证安装是否成功
```
sudo docker --version
```

## 1.3 Docker配置
对于通过systemd来管理服务的系统(比如CentOS7.X、Ubuntu16.X), Docker有两处可以配置参数: 一个是docker.service服务配置文件,一个是Docker daemon配置文件daemon.json。

1. docker.service  
对于CentOS系统，docker.service默认位于/usr/lib/systemd/system/docker.service；对于Ubuntu系统，docker.service默认位于/lib/systemd/system/docker.service

2. daemon.json  
daemon.json默认位于/etc/docker/daemon.json，如果没有可手动创建，基于systemd管理的系统都是相同的路径。通过修改daemon.json来改过Docker配置，也是Docker官方推荐的方法。

**以下说明均基于systemd,并通过/etc/docker/daemon.json来修改配置。**

### 1.3.1 配置镜像下载和上传并发数

从Docker1.12开始，支持自定义下载和上传镜像的并发数，默认值上传为3个并发，下载为5个并发。通过添加”max-concurrent-downloads”和”max-concurrent-uploads”参数对其修改。

编辑/etc/docker/daemon.json加入以下内容:

```json
"max-concurrent-downloads": 3,
"max-concurrent-uploads": 5
```

### 1.3.2 配置镜像加速地址

Rancher从v1.6.15开始到v2.x.x,Rancher系统相关的所有镜像(包括1.6.x上的K8S镜像)都托管在Dockerhub仓库。Dockerhub节点在国外，国内直接拉取镜像会有些缓慢。为了加速镜像的下载，可以给Docker配置国内的镜像地址。

编辑/etc/docker/daemon.json加入以下内容：

```json
"registry-mirrors": ["https://registry.docker-cn.com"],
```

如果是再阿里云上部署，则内容如下：
```
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://on68epbi.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload && systemctl restart docker
```

> 可以设置多个registry-mirrors地址，以数组形式书写，地址需要添加协议头(https或者http)。

### 1.3.3 配置私有仓库

Docker默认只信任TLS加密的仓库地址(https)，所有非https仓库默认无法登陆也无法拉取镜像。insecure-registries字面意思为不安全的仓库，通过添加这个参数对非https仓库进行授信。可以设置多个insecure-registries地址，以数组形式书写，地址不能添加协议头(http)。

编辑/etc/docker/daemon.json加入以下内容：
```
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries":["172.18.3.108"]
}
EOF
systemctl daemon-reload && systemctl restart docker
```

### 1.3.4 配置Docker存储驱动

OverlayFS是一个新一代的联合文件系统，类似于AUFS，但速度更快，实现更简单。Docker为OverlayFS提供了两个存储驱动程序:旧版的overlay，新版的overlay2(更稳定)。

先决条件:

* overlay2: Linux内核版本4.0或更高版本，或使用内核版本3.10.0-514+的RHEL或CentOS。
* overlay: 主机Linux内核版本3.18+
* 支持的磁盘文件系统
  - ext4(仅限RHEL 7.1)
  - xfs(RHEL7.2及更高版本)，需要启用d_type=true。 
  
> 具体详情参考 [Docker Use the OverlayFS storage driver](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)


编辑/etc/docker/daemon.json加入以下内容：

```json
"storage-driver": "overlay2",
"storage-opts": ["overlay2.override_kernel_check=true"]
```

### 1.3.5 配置日志驱动

容器在运行时会产生大量日志文件，很容易占满磁盘空间。通过配置日志驱动来限制文件大小与文件的数量。 >限制单个日志文件为100M,最多产生3个日志文件.

编辑/etc/docker/daemon.json加入以下内容：

```json
"log-driver": "json-file",
"log-opts": {
    "max-size": "100m",
    "max-file": "3"
    }
```