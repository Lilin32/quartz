---
title: git 泄露
tags:
  - 渗透测试
  - git
creation date: 2024-03-31
done: true
---
> [!note]
> Git 泄露通常是指 `.git` 文件夹泄露，可能会导致敏感信息暴露。`.git` 文件夹包含了项目的历史记录，包括提交、分支、tag 等信息。如果不小心泄露，可能会对项目的安全造成威胁。  
> 参考内容：  
> [A Hacker's Guide to Git | Wildly Inaccurate](https://wildlyinaccurate.com/a-hackers-guide-to-git/)  


# 1. .git 目录结构  
---
- **HEAD**：指向当前活动分支的引用。它是一个指针，指向当前分支的最新提交。  
- **config**：存储仓库的配置信息，如远程仓库的 URL、用户名和邮箱等。  
- **description**：用于 GitWeb 等工具显示仓库的描述信息。  
- **index**：包含了暂存区（stage）的内容，记录了即将提交的文件和相关元数据。  
- **hooks 文件夹**：包含一系列的钩子脚本，这些脚本可以在 Git 的各个阶段（如提交、变基、拉取等）自动执行。  
- **info 文件夹**：包含一些辅助性的信息。  
- **objects 文件夹**：存储所有的 Git 对象，包括提交（commit）、树（tree）和 blob 对象。这些对象被压缩并存储在该目录下。内容经过 **zlib** 压缩。  
	- **pack**：包含打包的对象数据。
- **refs 文件夹**：包含指向提交的引用，如分支（branches）和标签（tags）。
- **logs 文件夹**：存储了每个引用（分支、标签等）的修改历史。  

在 `.git/objects` 目录下，每个对象都被存储为一个以 40 个字符的 SHA-1 值命名的文件，前两个字符作为目录名，后 38 个字符作为文件名。  
在 git 中，每个对象都可以通过唯一的 SHA-1 值来引用和访问。  

# 2. index 文件  
---
> [!tip]
> 参考内容：  
> [Git - index-format Documentation](https://git-scm.com/docs/index-format)    
> [DevOps - Git Internals: Architecture and Index Files | Microsoft Learn](https://learn.microsoft.com/en-us/archive/msdn-magazine/2017/august/devops-git-internals-architecture-and-index-files)    
> [Git: Understanding the Index File - Mincong Huang](https://mincong.io/2018/04/28/git-index/#23-extensions)

存储路径：`.git/index`  
1. **head** 部分：共 12 字节  
	- 4 字节签名：通常是 `DIRC`，表示 dircache  
	- 4 字节版本号：目前支持的版本有 2、3、4  
	- 4 字节的索引条目数量  
2. **index entry** 部分：
	- 8 字节的 ctime：4 字节的文件元数据最后修改时间，4 字节的纳秒分数  
	- 8 字节的 mtime：4 字节的文件数据最后修改时间，4 字节的纳秒分数  
	- 4 字节的 dev：文件的设备 ID  
	- 4 字节的 ino：文件索引节点号（inode number）  
	- 4 字节的 mode：文件权限，将 16 进制内容转换为 8 进制内容，即可获得常见的文件权限格式，例如：`100644`  
	- 4 字节的 uid  
	- 4 字节的 gid  
	- 4 字节的文件大小  
	- 20 字节的 **SHA-1 Object ID**  
	- 2 字节的 flag：在 git 版本为 2 和 3 及以上版本中，这两个字节的内容的含义有所区别  
	- 文件路径  
	- 空字节，将整个的 index entry 部分填充为 8 字节的整数倍，并保持最终以 NULL 结尾  
3. **Extensions** 扩展部分  
	1. 缓存树扩展 **head** 部分
	2. 缓存树扩展 **entry** 部分
4. 索引校验和

# 3. pack 包文件  
---
>[!tip]
> `pack` 文件夹是高效存储和检索 Git 对象的关键组件。 Git 使用 packfile 来压缩和存储对象，这可以显着减少存储存储库所需的磁盘空间量。  
> 参考内容：  
> [Git - pack-format Documentation](https://git-scm.com/docs/pack-format/2.37.0)  
> [Git Packfiles](https://kennyballou.com/blog/2017/03/git-packfiles/index.html)  

存储路径：`.git/objects/pack`  
包文件有两种文件：`.pack` 和 `.idx`  

## 3.1 .idx 文件  
1. 4 字节的 magic number ：固定为 `FF 74 4F 63`  
2. 4 字节的版本号
3. 4 字节的网络字节序整数，256 个 ，字节序的范围是：`0008h ~ 0407h`，并且最后 4 字节是对象数量。可以和 `.pack` 文件的第 3 个 4 字节的对象数量对应。  
4. 20 字节的 object name，n 个，已排序  
5. 4 字节的 object entry 的 CRC32 校验值，n 个  
6. 4 字节的 object entry 在 `.pack` 文件中的偏移值，n 个
7. 8 字节的偏移条目，若 pack 文件小于 2 GB 则为空  
8. 20 字节的 `.pack` 文件校验和  
9. 20 字节的 `.idx` 所有内容的校验和  

## 3.2 .pack 文件  
1. **head** 部分：12 字节
	- 4 字节的签名：通常是 `PACK`  
	- 4 字节版本号
	- 4 字节的对象数量  
2. **object entry** 部分  
	- 类型：将内容转换为 2 进制，第一位是 **MSB**，后三位是 **type**  
		- commit：1 （001）  
		- tree：2 （010）  
		- blob：3 （011）  
		- tag：4 （100）  
		- ofs_delta：6 （110）  
		- ref_delta：7 （111）  
	- 长度：该部分是一个变长的内容，使用每个字节的七个最低有效位来形成结果整数。只要最高有效位为 1，这个过程就会继续，最终这些 7 bit 内容被连接起来  
	- representation  
		- 以 78 （十六进制）开头的内容为使用 zlib 压缩的内容  
1. 最后 20 字节的**校验和**：head 部分以及所有 object entry 的校验和。  

# 4. .git 泄露利用工具  
---
[GitHub - gakki429/Git\_Extract: 提取远程 git 泄露或本地 git 的工具](https://github.com/gakki429/Git_Extract)  
[GitHub - lijiejie/GitHack: A \`.git\` folder disclosure exploit](https://github.com/lijiejie/GitHack)  

