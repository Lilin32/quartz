---
title: Linux 扫盲
tags:
  - Linux
creation date: 2024-03-14
done: true
---
# 1. kali 虚拟机初始配置  
---
## 1.1 修改 root 用户密码  
```Shell
sudo passwd root
```

## 1.2 Linux 的网络配置  
### 1.2.1 配置 IP 地址  
配置路径：`/etc/network/interfaces`  

```Shell
auto <ens-name>
iface <ens-name> inet static
address <address>      # IP地址
gateway <gateway>      # 网关
netmask <netmask>      # 子网掩码
broadcast <broadcast>  # 广播

auto <ens-name>
iface <ens-name> inet DHCP
```
### 1.2.2 配置 DNS
配置路径：`/etc/resolv.conf`
添加 `name <IP>`，即可添加 DNS 服务器

### 1.2.3 重启网络服务
```Shell
systemctl restart networking
systemctl restart nreworking.service
dhclient # 自动获取IP
```
## 1.3 设置 apt 源
配置路径：`/etc/apt/sources.list`
1. 清华大学镜像站：[清华大学 TUNA 协会](https://tuna.moe/)
	- [kali | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/help/kali/)
	- [ubuntu | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)
2. 中科大镜像站：[USTC Open Source Software Mirror](https://mirrors.ustc.edu.cn/)
	- [Kali Linux 源使用帮助 — USTC Mirror Help 文档](https://mirrors.ustc.edu.cn/help/kali.html)
	- [Ubuntu 源使用帮助 — USTC Mirror Help 文档](https://mirrors.ustc.edu.cn/help/ubuntu.html)
3. 阿里云镜像站：[阿里巴巴开源镜像站-OPSX镜像站-阿里云开发者社区](https://developer.aliyun.com/mirror/)
	- [kali镜像\_kali下载地址\_kali安装教程-阿里巴巴开源镜像站](https://developer.aliyun.com/mirror/kali?spm=a2c6h.13651102.0.0.b98b1b11hXrY3p)
	- [ubuntu镜像\_ubuntu下载地址\_ubuntu安装教程-阿里巴巴开源镜像站](https://developer.aliyun.com/mirror/ubuntu?spm=a2c6h.13651102.0.0.3e221b1135P6lu)

```Shell
apt-get update    # 更新软件包数据库
apt-get upgrade   # 升级已安装的软件包
apt-get clean     # 清除 apt 缓存
apt-get full-upgrade -y    # 更新到最新发行版
```

## 1.4 配置 python
### 1.4.1 下载 pip
Kali 已经默认安装了 python2.7 和 python3.9，需要手动安装 pip，如果需要可以配置国内的 pip 下载源。
```Shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip-2.py

python2 get-pip-2.py
python3 get-pip.py

pip2 list
pip3 list
```

### 1.4.2 配置 pip 国内镜像源
[pypi | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/help/pypi/)
```Shell
# 以 pip 方式设置默认源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 以 pdm 方式设置默认源
pdm config pypi.url https://pypi.tuna.tsinghua.edu.cn/simple
```

## 1.5 配置 ssh
ssh 是 secure shell 的缩写，是建立在应用层基础上的安全协议。为远程登录会话和其他网络服务提供安全性的协议。
配置文件路径：`/etc/ssh/sshd_config`
```Shell
# 开启允许root权限登录
PermitRootLogin yes
# 开启密码认证
PasswordAuthentication yes

# 开启ssh服务
service ssh start
# 重启ssh服务
service ssh restart

# 添加开机启动
update-rc.d ssh enable
```

# 2. Linux 目录结构
---
可以通过`apt install tree`下载软件，在`/`路径下执行`tree -d -L 1`（`-d`：仅查看文件夹；`-L`：设置层级）

![Linux 目录结构](https://image.yiscook.top/blog-image/202403150205018.png)

| **目录**    | **说明**                                                   |
| --------- | -------------------------------------------------------- |
| /bin      | Binary的缩写，存放经常使用的命令                                      |
| /boot     | 存放Linux启动时候的核心文件，包括连接文件及镜像文件                             |
| /dev      | Device的缩写，存放Linux的外部设备，在Linux中访问设备的方式和访问文件的方式是相同的，万物基于文件 |
| /etc      | 存放所有系统管理所需要的配置文件和子目录                                     |
| /home     | 每个用户都有自己的目录，一般以用户的账户命名                                   |
| /root     | 系统管理员目录                                                  |
| /sbin     | 存放系统管理员使用的系统管理程序                                         |
| /usr      | 存放用户的应用程序和文件等，类似Windows中的program file目录                  |
| /usr/bin  | 系统中用户使用的应用程序                                             |
| /usr/sbin | 管理员使用的管理程序和系统守护程序                                        |
| /tmp      | 存放临时文件；注意：任何用户都可以进行读写执行的权限                               |
| /lib      | 存放系统最基本的动态连接共享库，类似Windows中的DLL文件。几乎所有应用程序都需要用到           |
| /var      | 存放不断扩充的内容，习惯将经常被修改的目录或文件存放到该目录下，例如日志等                    |

# 3. Linux 文件属性
---  

| 文件类型 | 属主权限  | 属组权限  | 其他用户权限 |
| :--: | :---: | :---: | :----: |
|  0   | 1 2 3 | 4 5 6 | 7 8 9  |
|  d   | r w x | r - x | r - x  |
| 目录文件 | 读写执行  | 读写执行  |  读写执行  |


使用的命令：`chmod`、`chown`
可以使用数字设置对应的权限：**r → 4**；**w → 2**；**x → 1**
`chown`：需要使用管理员权限才能执行此命令，非管理员用户可以使用 `chgrp` 来设置关联组

# 4. Linux 常用基本命令
---

| 命令        | 描述                     | 用法                                                                                                          |
| --------- | ---------------------- | ----------------------------------------------------------------------------------------------------------- |
| **mkdir** | 用于创建目录                 | `mkdir [-p] dirName`<br>`-p`：确保目录名称存在，不存在则创建一个。<br>例如：`mkdir -p test/testDir`不会报错，可以成功创建。                   |
| **rmdir** | 删除空的目录                 | `rmdir [-p] dirName`<br>`-p`：当子目录被删除后，使得dirName成为空目录，一并进行删除。                                                |
| **cp**    | 复制文件或目录                | `cp [options] source... directory`<br>`-r`：可以将目录下的所有文件复制到新目录下。                                              |
| **rm**    | 删除一个文件或目录              | `rm [options] name...`<br>`-i`：每次删除前进行询问；<br>`-f`：忽略不存在的文件和参数，删除前不进行确认；<br>`-r`：递归删除目录及其内容，经常会配合使用：`rm -rf` |
| **mv**    | 用来对文件或目录重新命名，或者将文件进行移动 | 如果将一个文件移到一个已经存在的目标文件中，则目标文件的内容将会被覆盖。  <br>`mv`类似文件“搬家”，文件个数并未增加；`cp`对文件进行复制，文件个数增加。                         |

# 5. Vim 编辑器
---
[GitHub - HanielF/VimTutor: VimTutor 中英对照版，给所有想入门Vim的Vimer](https://github.com/HanielF/VimTutor)
