---
title: Docker 安装指南
description: 全面的 Docker 安装指南，涵盖 Linux 各类发行版，以及一些生产化的配置
published: 2026-01-01
date: 2026-01-01
tags:
  - Docker
category: 教程
draft: false
pinned: false
image: ./cover/docker-install.png
---
> [!IMPORTANT]
> PS：此文章为归档文章，日期固定为 2026-01-01，如有修改将修改为最后修改日期，以便重新启用该文章


> [!NOTE]
> 因为 Docker 镜像在国内拉取较慢，所以我们一般使用镜像站进行拉取，或者自建 Docker 仓库
> 
> 这里推荐一个镜像站[轩辕镜像站](https://xuanyuan.cloud)

# 安装 Docker

## 一行命令安装（推荐）

专为国内用户优化的 Docker 镜像加速解决方案，一键配置即可享受极速下载体验

```bash
bash <(curl -sSL https://xuanyuan.cloud/docker.sh)
```

## Ubuntu 手动安装

```bash
# 安装前先卸载操作系统默认安装的docker，
sudo apt-get remove docker docker-engine docker.io containerd runc

# 安装必要支持
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release

# 添加 Docker
## 官方 GPG key （可能国内现在访问会存在问题）
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
## 阿里源（推荐使用阿里的gpg KEY）
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 添加 apt 源:
## Docker官方源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
## 阿里apt源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 更新源
sudo apt update
sudo apt-get update

# 安装最新版本的Docker
sudo apt install docker-ce docker-ce-cli containerd.io

# 等待安装完成...

# 查看Docker版本
sudo docker version

# 查看Docker运行状态
sudo systemctl status docker
```

## CentOS / RHEL

```bash
# 更新 yum 包
yum -y update
## 区分
## yum -y update：升级所有包同时也升级软件和系统内核；
## yum -y upgrade：只升级所有包，不升级软件和系统内核

# 卸载旧版本
yum remove docker  docker-common docker-selinux docker-engine

# 安装需要的软件包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 设置 yum 源
## 设置一个yum源，下面两个都可用
## 中央仓库
yum-config-manager --add-repo http://download.docker.com/linux/centos/docker-ce.repo
## 阿里仓库 推荐
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 选择docker版本并安装
## 查看可用版本
yum list docker-ce --showduplicates | sort -r
## 安装指定版本
#sudo yum -y install docker-ce-17.12.1.ce
## 安装Docker
yum -y install docker-ce

# 等待安装完成...

# 查看Docker版本
docker version

# 查看Docker运行状态
systemctl status docker
```

## Alpine

```bash
# 更新软件源 确保有community源
apk update

# 安装docker
apk add docker docker-compose

# 配置文件
## 创建配置目录
mkdir -p /etc/docker
## 配置daemon.json
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "5m",
    "max-file": "2"
  },
  "storage-driver": "overlay2",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 1024,
      "Soft": 512
    }
  },
  "max-concurrent-downloads": 1,
  "max-concurrent-uploads": 1
}
EOF

## 重启Docker使配置生效
rc-service docker restart

# 常用命令
## 开机自启
rc-update add restart boot
## 重启Docker服务
rc-service docker restart
## 启动Docker服务
rc-service docker start
## 停止Docker服务
rc-service docker stop
## 查看Docker服务状态
rc-service docker status
```

# 生产环境配置 Docker

## 更换工作目录

由于docker默认工作目录在 `/var/lib/docker/` 下，生产环境的系统磁盘有限，所以我们需要将默认目录修改到自定义的目录下，本文修改到 `/opt/docker/` 目录下

**注意：此操作可能会造成Docker数据丢失，建议在刚安装完docker后进行此操作**

原位置：`/var/lib/docker/`

**修改配置**

```bash
# 停止DOcker服务
sudo systemctl stop docker

# 创建新的数据目录，用于存储Docker数据
mdkir -p /opt/docker/

# 修改这个数据目录的权限
chmod -R 777 /opt/docker/

# 复制文件+权限 （如有）
sudo cp -a /var/lib/docker/* /opt/docker/

# 创建Docker配置文件
mkdir /etc/docker
vim /etc/docker/daemon.json
## 在/etc/docker/daemon.json 文件内添加

{
  "data-root": "/opt/docker"
}

# 如果需要转移数据，此操作必须在转移数据后操作！！！
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证是否成功
## 在输出的信息中，找到“Docker Root Dir”一项。它应该显示新的存储路径，例如 /opt/docker/。
sudo docker info

# 确认数据无误后，删除原来的内容（可选）
rm -rf /var/lib/docker
```

## 限制日志数量

**修改配置文件（被动）**

给 Docker 添加配置文件，限制日志文件数量及其大小。

```bash
# 修改 Docker 配置文件
vim /etc/docker/daemon.json

# 添加以下配置

"log-opts": {"max-size":"500m", "max-file":"3"}

# 重载并重启docker服务
systemctl daemon-reload
systemctl restart docker
```

> max-size=500m，意味着一个容器日志大小上限是500M
> 
> max-file=3，意味着一个容器有三个日志，分别是id+.json、id+1.json、id+2.json
> 
> 注：设置后只对新添加的容器有效。

**Shell 脚本删除 Docker 日志（主动）**

写一个脚本，定时运行进行删除 Docker 日志

```bash
#!/bin/bash

# Docker容器日志清理脚本 du -h --max-depth=1

# 设置Docker日志文件存储路径
log_path="/opt/docker/containers"

# 获取所有容器ID
container_ids=$(ls -1 $log_path)

# 循环处理每个容器
for container_id in $container_ids; do
    # 构造日志文件路径
    log_file="${log_path}/${container_id}/${container_id}-json.log"

    # 检查日志文件是否存在
    if [ -f "$log_file" ]; then
        echo "清理容器 ${container_id} 的日志文件: ${log_file}"
        
        # 清空日志文件
        truncate -s 0 "$log_file"
    else
        echo "未找到容器 ${container_id} 的日志文件: ${log_file}"
    fi
done

echo "日志清理完成。"
```

## 其他配置

```sql
# 指定私服仓库地址
"insecure-registries": [
  "http://harbor:30001"
]

# 镜像站点
"registry-mirrors": [
    "http://harbor:30001",
    "https:/docker.1panel.live"
  ]
```

# 签名证书，实现私服访问

## Linux

```bash
# 1. 获取Harbor服务器的证书
openssl s_client -connect harbor.chengyun.local:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > harbor-cert.pem

# 2. 创建Docker证书目录
sudo mkdir -p /etc/docker/certs.d/harbor.chengyun.local

# 3. 复制证书到Docker信任目录
sudo cp harbor-cert.pem /etc/docker/certs.d/harbor.chengyun.local/ca.crt

# 4. 同时添加到系统CA证书（可选但推荐）
sudo cp harbor-cert.pem /usr/local/share/ca-certificates/harbor.chengyun.local.crt
sudo update-ca-certificates

# 5. 重启Docker服务
sudo systemctl restart docker
```

## Windows

```bash
# 在git Bash中执行以下命令
echo | openssl s_client -connect harbor.chengyun.local:443 -servername harbor.chengyun.local 2>/dev/null | openssl x509 > harbor-ca.crt
```

然后把这个 `harbor-ca.crt` **作为根 CA 安装到“受信任的根证书颁发机构”**

1. 在文件资源管理器中双击 `harbor-ca.crt`
2. 点击 **“安装证书…”**
3. 选择 **“本地计算机”** → 下一步
4. 选择 **“将所有证书放入下列存储”** → 点击“浏览”
5. 选择 **“受信任的根证书颁发机构”** → 确定 → 下一步 → 完成
6. 输入管理员密码（如有）
7. 重启 Docker Desktop

# 备份/迁移 Docker

_先欠着_

# 卸载 Docker

> [!CAUTION]
> 在卸载前，需确认是否保留容器、镜像、卷或配置文件

## Ubuntu

**停止 Docker 相关服务**

```bash
# 停止Docker服务
sudo systemctl stop docker docker.socket containerd.service
 
# 确认服务已停止
sudo systemctl status docker  # 应显示“inactive”
```

**移除 DOcker 包**

```bash
# 卸载Docker包
sudo apt purge -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-scan-plugin
 
# 自动清理未使用依赖
sudo apt autoremove -y
```

**删除残留文件与目录**

```bash
# 删除核心数据目录（镜像、容器、卷等）
sudo rm -rf /var/lib/docker /var/lib/containerd
 
# 删除配置文件
sudo rm -rf /etc/docker /etc/default/docker
 
# 删除用户配置
rm -rf ~/.docker
 
# 删除日志文件
sudo rm -rf /var/log/docker /var/log/containerd
```

## CentOS / RHEL

**停止服务与进程**

```bash
sudo systemctl stop docker containerd
sudo systemctl disable docker containerd  # 禁止开机自启
```

**移除 Docker 包**

```bash
sudo yum remove -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo yum autoremove -y
```

**清理残留文件**

```bash
sudo rm -rf /var/lib/docker /var/lib/containerd /etc/docker ~/.docker
sudo rm -rf /usr/lib/systemd/system/docker.service /usr/lib/systemd/system/docker.socket
sudo systemctl daemon-reload  # 刷新systemd配置
```

---

> [!IMPORTANT]
> PS：此文章为归档文章，日期固定为 2026-01-01，如有修改将修改为最后修改日期，以便重新启用该文章