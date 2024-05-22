---
title: sqlmap 源码解读：初识
tags:
  - 渗透测试
  - web安全
  - python
  - SQL注入
  - sqlmap
creation date: 2024-05-01
done: true
---
> [!tip]
> 中文手册：[文档介绍 | sqlmap 用户手册](https://sqlmap.highlight.ink/)  
> 官网：[sqlmap: automatic SQL injection and database takeover tool](https://sqlmap.org/)  
> 源码解读基于的 sqlmap

# 1. 目录结构  
---
- data：  
	- html：图形化界面内容  
	- procs：sqlmap 在目标系统上使用的 SQL 片段。包含了：mssqlserver，mysql，oracle，postgresql  
	- shell：包含 backdoor 和 stager 两个类型的恶意脚本，并且都进行了加密，此处的所有脚本都可以使用 `extra/cloak/cloak.py` 脚本进行加密和解密  
	- txt  
	- udf：存储了 mysql 和 postgresql 的 udf 二进制文件，是用于目标系统的文件。此部分源码在：[GitHub - sqlmapproject/udfhack: Database takeover UDF repository](https://github.com/sqlmapproject/udfhack)  
	- xml  
- doc：各种语言版本的 README 文档及其他文档内容  
- extra：一些额外功能  
	- beep：蜂鸣器，用于在不同系统上播放蜂鸣  
	- cloak：简易的文件加解密、解压缩工具，可对 `data/shell` 中的脚本进行解密  
	- dbgtool：主要用于特定的转换需求，将二进制文件转换为可以在 DEBUG. exe 中使用的 ASCII 脚本  
	- icmpsh：icmp 协议的反弹 shell 脚本。包含了 exe 可执行文件，c，perl，python 编写的脚本  
	- runcmd：辅助程序，用于运行命令提示符命令  
	- shellcodeexec：用于在内存中执行恶意代码，此部分源码在：[GitHub - bdamele/shellcodeexec: Script to execute in memory a sequence of opcodes](https://github.com/bdamele/shellcodeexec)  
	- shutils：存放一些 sh 的工具脚本  
	- vulnserver：python 编写的模拟 SQL 注入漏洞的服务  
- lib：检测、执行 SQL 注入的部分，需要重点学习  
- plugins
	- dbms：针对不同种类数据库，连接数据库，执行查询，获取数据库相关信息，通过数据库进行文件操作，识别数据库指纹，实现 RCE 攻击等功能  
	- generic：dbms 的通用父类
- tamper：用于绕过一些安全措施的脚本  
- thirdparty：sqlmap 所使用的第三方库
- sqlmap.conf：sqlmap 配置文件  
- sqlmap.py：sqlmap 主文件  
- sqlmapapi.py：sqlmap 的 api 文件，可将 sqlmap 部署到 web 应用  
- sqlmapapi.yaml：sqlmap 的 api 文档，描述各个接口的功能和使用方法  

# 2. sqlmap.py (入口文件)  
---
![sqlmap.py](https://image.yiscook.top/blog-image/202405031752024.png)  
## 2.1 try except 部分  
1. try 部分：主要进行了一系列的模块导入操作，并且调整警告消息的输出。  
2. except 部分：用于处理键盘输入的中断异常（ctrl + c），并给出响应的退出信息。  

## 2.2 modulePath 函数  
文件所在目录的绝对路径，并使用 getUnicode 函数将路径转换为 Unicode 编码。  

## 2.3 checkEnvironment 函数  
检查运行环境是否满足要求。
1. 首先检查模块路径是否为目录，如果不是则抛出异常。  
2. 检查当前运行的版本是否低于 1.0，如果是则抛出异常。  
3. 如果是通过 pip 安装的环境，则对一些全局变量进行修补。  

## 2.4 main 函数  
这个 Python 函数是一个命令行工具的主函数，主要负责处理命令行参数并执行相应的操作。函数首先进行一些初始化工作，如解析命令行参数、设置环境变量等。然后根据命令行参数的不同，执行不同的操作，包括爬取目标 URL、进行漏洞测试等。  
```python
def main():  
    """  
    Main function of sqlmap when running from command line.    
    """  
    
    try:  
        dirtyPatches()  
        resolveCrossReferences()  
        checkEnvironment()  
        setPaths(modulePath())  
        banner()
```  
此部分主要负责以下内容：  
1. 进行了一些补丁设置；  
2. 解决跨引用问题；  
3. 检测环境变量是否满足运行要求；  
4. 项目执行时，设置 sqlmap 项目目录及各个文件的绝对路径，以及检测 txt 和 xml 文件是否存在；
5. 打印 sqlmap 的 banner 以及版本信息。

### 2.4.1 输入命令解析并初始化  
```python
# Store original command line options for possible later restoration  
args = cmdLineParser()  
cmdLineOptions.update(args.__dict__ if hasattr(args, "__dict__") else args)  
initOptions(cmdLineOptions)
```
1. 解析终端中输入的命令，并保存在 args 中；  
2. 根据 args 更新全局变量 cmdLineOptions；  
3. 根据 cmdLineOptions 初始化 option。  

- `cmdLineParser()`  
	- 位置：lib.parse.cmdline.cmdLineParser  
	- 该函数主要使用到 ArgumentParser 类（是 Python 标准库 argparse 模块的一个核心类。它的主要作用是解析命令行参数和选项。）    
	- 主要功能如下：  
		1. 定义参数，具体内容可参考：[用法 | sqlmap 用户手册](https://sqlmap.highlight.ink/usage)  
		2. 对命令行参数解析器（parser）进行修改  
		3. 对输入的命令行参数进行预处理  
		4. 使用 parser 对象解析参数，并尝试处理解析过程中可能出现的异常。  
		5. 返回解析后的 args 对象  
- `initOptions()`  
	- 位置：lib.core.option.initOptions
	- 初始化选项信息，包括全局变量中的 conf（配置信息）、kb（知识库）、mergedOptions（合并后的选项）

```python
init()
```
- 位置：lib.core.option.init  
- 该函数根据命令行和配置文件选项设置配置和知识库单例的属性。函数内部调用了多个私有函数，执行了一系列的初始化操作，

### 2.4.2 开始 SQL 注入  
```python
# 根据配置决定是否更新所有项目
if not conf.updateAll:  
    # 延迟导入（加快启动速度）
    if conf.smokeTest:  
	    # 执行冒烟测试，并根据测试结果设置退出码
		……
    elif conf.vulnTest:  
	    # 执行漏洞测试，并根据测试结果设置退出码
        …… 
    else:  
	    # 正常启动流程
        from lib.controller.controller import start  
        if conf.profile:  
	        # 如果启用了性能分析，导入性能分析工具并执行
            from lib.core.profiling import profile  
            globals()["start"] = start  
            profile()  
        else:  
            try:  
	            # 根据配置进行深度爬取或直接启动
                if conf.crawlDepth and conf.bulkFile:  
                    targets = getFileItems(conf.bulkFile)  
                    for i in xrange(len(targets)):  
                        target = None  
                        try:  
                            …… 
                        except Exception as ex:  
                            ……  
                        else:  
                            if kb.targets:  
                                start()  
                else:  
                    start()  
            except Exception as ex:  
                ……
```
- `start()`：
	- 位置：lib.controller.controller.start  
	- 用于检测给定 URL 是否易受 SQL 注入攻击。如果发现注入点，函数会记录并报告给用户。  
	- 详细内容可以查看：[[sqlmap 源码解读：controller.py#1. start 函数]]

### 2.4.3 异常处理  
1. SqlmapUserQuitException：当用户退出程序时，记录错误信息；  
2. SqlmapSilentQuitException 和 bdb.BdbQuit：这两种异常会让程序静默退出，不做任何处理；  
3. SqlmapShellQuitException：当退出 sqlmap shell 时，设置 sqlmap shell 的标志为 False；  
4. SqlmapBaseException：捕获所有 sqlmap 相关的基础异常，记录错误信息并退出程序；   
5. KeyboardInterrupt：当程序被用户中断时（例如按下 Ctrl+C），尝试打印一个空行，然后记录错误信息并退出程序；  
6. EOFError：当程序读取到 EOF（文件结束）时，打印一个空行，记录错误信息并退出程序；  
7. SystemExit：当调用 sys.exit ()退出程序时，设置退出码并退出程序；  
8. 其他所有异常：打印一个空行，记录未处理的异常信息，并退出程序；  

## 2.5 程序入口管理  
尝试执行 main 函数，包括了其中的异常处理以及退出逻辑。  

# 3. sqlmap 全局变量  
---
路径：`lib/core/data.py`  
```python
#!/usr/bin/env python

"""
Copyright (c) 2006-2024 sqlmap developers (https://sqlmap.org/)
See the file 'LICENSE' for copying permission
"""

# 导入自定义数据类型和日志记录器
from lib.core.datatype import AttribDict
from lib.core.log import LOGGER

# 定义 sqlmap 路径的存储对象
paths = AttribDict()

# 存储原始命令行选项的对象
cmdLineOptions = AttribDict()

# 存储合并后的选项（命令行、配置文件和默认选项）的对象
mergedOptions = AttribDict()

# 存储配置信息 configuration
conf = AttribDict()

# 存储知识库 knowledge base
kb = AttribDict()

# 存储每个数据库管理系统特定查询的字典
queries = {}

# 日志记录器
logger = LOGGER
```
