---
title: Claw VPS 系统配置笔记
description: 记录购买 Claw VPS 后的系统配置过程，包括硬件配置、系统选择、创建普通用户并授予 sudo 权限、配置 SSH 登录、安装防火墙以及增加 swap 分区等步骤。
tags: [VPS, ArchLinux, SSH, UFW, Swap]
pubDate: 2025-01-23
draft: false
---

最近趁着活动，购买了 Claw VPS，折腾了一番，记录一下。

## 硬件配置

- 1 CPU
- 1 GB RAM
- 20 GB SSD
- 500 GB 流量（200Mbps）
- 1 IPv4
- 1 IPv6
- 价格：首年 7 刀
- 地区：日本

## 系统选择

系统选择 ArchLinux 系统，主要是因为软件包比较多，而且更新比较快，默认安装的软件包也比较少，系统占用少。

## 创建普通用户并授予 sudo 权限

第一步应该创建一个非 root 用户，然后限制 root 用户登录。

通过 ssh 连接到服务器后发现默认已经创建了 Arch 的用户，所以只需要对其进行配置即可。

1. 创建用户（不需要，默认已经创建了一个 Arch 用户）

```bash
useradd -m -G wheel -s /bin/bash arch
```

1. 设置用户密码

使用 passwd 命令为新用户设置密码：

```bash
sudo passwd arch
```

系统会提示你输入并确认新密码。

1. 配置 sudo 权限(不需要，默认 Arch 用户已经配置了 sudo 权限)

确保 wheel 组的成员可以使用 sudo。编辑 sudoers 文件：

```bash
visudo
```

找到以下行：

```bash
# %wheel ALL=(ALL) ALL
```

去掉行首的 # 注释符号，使其生效：

```bash
%wheel ALL=(ALL) ALL
```

保存并退出编辑器。

1. 验证

切换到新用户并验证 sudo 权限：

```bash
su - arch
sudo ls /root
```

如果系统提示输入密码并成功执行命令，说明 sudo 权限已正确配置。

## 配置 SSH 登录

1. 使用 ssh-keygen 命令生成 SSH 密钥对：

```bash
ssh-keygen -t rsa -b 4096
```

1. 将公钥添加到服务器：

   将你的 SSH 公钥添加到用户的 ~/.ssh/authorized_keys 文件中：

```bash
mkdir -p ~/.ssh
sudo chmod 700 ~/.ssh
echo "你的公钥内容" | sudo tee -a ~/.ssh/authorized_keys
sudo chmod 600 ~/.ssh/authorized_keys
```

1. 配置 SSH 服务器：

编辑 SSH 服务器配置文件 /etc/ssh/sshd_config，设置以下内容：

```bash
PermitRootLogin no          # 禁止 root 用户登录
PubkeyAuthentication yes    # 启用公钥认证
PasswordAuthentication no   # 禁止密码登录
```

保存并退出编辑器后，重启 SSH 服务以应用更改：

```bash
sudo systemctl restart sshd
```

由于 claw 安装的 archlinux 默认配置了一些 ssh 参数，所以需要一些额外的操作：

查看/etc/ssh/sshd_config.d/文件夹下的内容，删除或注释掉其中的配置文件，然后重启 ssh 服务。

## 安装防火墙

1. 安装防火墙：

```bash
sudo pacman -S ufw
```

1. 启用防火墙：

```bash
sudo ufw enable
```

1. 配置防火墙规则：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit ssh
```

## 增加 swap 分区

1. 首先查看原 swap 信息

```bash
sudo swapon --show
```

1. 关闭原 swap

```bash
sudo swapoff /swap/swapfile
```

1. 创建新 Swap 文件

首先，创建一个新的 Swap 文件。例如，创建一个 2GB 的 Swap 文件：

```bash
sudo fallocate -l 2G /swap/swapfile
```

1. 设置权限
   确保 Swap 文件的权限正确：

```bash
sudo chmod 600 /swapfile
```

1. 格式化 Swap 文件

```bash
sudo mkswap /swapfile
```

1. 启用 Swap 文件

```bash
sudo swapon /swapfile
```

1. 永久生效
   为了在系统重启后自动启用 Swap 文件，编辑 `/etc/fstab` 文件，添加以下内容：

```bash
/swapfile none swap sw 0 0
```
