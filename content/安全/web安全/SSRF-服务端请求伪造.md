---
title: SSRF-服务端请求伪造
tags:
  - web安全
creation date: 2024-03-17
done: false
---
# 1. 概述  
---
>[!note]
>**服务端请求伪造**（Server-side Request Forgery，SSRF），攻击者利用服务器端应用程序对远程服务器进行请求，将请求伪装成来自于被攻击服务器本身的请求，从而获取目标服务器的敏感信息或者攻击内网的服务器。SSRF 攻击通常利用应用程序提供的一些功能（例如：URL 参数、请求数据、文件上传等）来构造攻击请求。  
## 1.1 形成原因  
一般情况下，攻击者是无法从外网访问到内部系统。漏洞形成的主要原因是服务端提供了从其他服务器应用获取数据的功能，但是没有对目标地址做严格的过滤与限制，导致攻击者可以传入任意的地址来让后端服务器对其他服务器发送请求，并返回对请求的数据。  

## 1.2 危害  
可以获取 web 应用可达服务器服务的 banner 信息，以及收集内网 web 应用的指纹信息。可以利用这些信息进行下一步渗透。  
攻击运行在内网的系统火应用程序，获取内网系弱口令进行内网漫游。对有漏洞的内网 web 应用进行攻击。  

## 1.3 常见的漏洞发现点  
- 通过 URL 地址分享网页内容  
- 文件处理、编码处理、转码等服务：通过 URL 地址对其他内容进行调整  
- 通过 URL 地址加载或者下载图片  
- 图片、文章等收藏功能  

# 2. 相关危险函数  
---
## 2.1 PHP  
- `file_get_contents()`：该函数可用于读取远程 URL 的内容，可能导致 SSRF 漏洞。  
- `curl_exec()`：当传入的 URL 参数不经过验证或者校验时，攻击者可以构造恶意 URL 进行利用。  
- `fsockopen()`：当传入的 URL 参数未经过校验时，攻击者可以构造恶意 URL 进行利用。  
- `simplexml_load_file()`：该函数可以解析 XML 文件，也可以读取远程的 XML 文件，可能导致 SSRF 漏洞。  
- `get_headers()`：该函数可以获取 URL 的响应头，可能导致 SSRF 漏洞。  

## 2.2 Java  
- 仅支持`HTTPS/HTTP`协议的类：`HttpClient`类、`HttpURLConnection`类、`OkHttp`类、`Request`类。  
- 支持`sun.net.www.protocol`所有协议的类：`URLConnection`类、`URL`类、`ImageIO`类  

## 2.3 python  
- `urllib2.urlopen()`：可以读取 URL 的内容，如果 URL 参数不经过验证或校验，可能导致 SSRF 漏洞。  
- `httplib.HTTPConnection`：可以创建 HTTP 连接并发送请求，如果 URL 参数不经过验证或校验，可能导致 SSRF 漏洞。  
- `xml.etree.ElementTree.parse()`：可以解析 XML 文件，如果 URL 参数不经过验证或校验，可能导致 SSRF 漏洞。  
- `socket.create_connection()`：可以创建套接字连接并发送请求，如果 URL 参数不经过验证或校验，可能导致 SSRF 漏洞。  

# 3. SSRF 类型及发现  
---
## 3.1 有回显  
通过目标服务器请求 url，如果返回了网站的信息，存在服务端请求伪造。  

## 3.2 无回显  
比较可惜，无回显的场景不能被利用。  
1. 需要借助DNSlog平台，帮助进行测试。  
[DNSLog Platform](http://www.dnslog.cn/)  
[DNSLOG Platform](https://dig.pm/)  
2. 使用 burpsuite 的 burp collaborator client，辅助进行测试  
简单粗暴的理解 SSRF 和 ur l 跳转漏洞：URL 跳转是直接跳转到一个页面。SSRF 是基于现有的页面，去加载另外一个页面。  

# 4. SSRF漏洞利用  
---
## 4.1 利用方法  
### 4.1.1 file 协议  
主要用于访问服务器本地中的文件。访问敏感信息的文件。  
```HTML
file://<file-path>  
```

### 4.1.2 dict 协议  
用于在互联网上查询单词的定义和相关信息。它通常用于在线词典服务和类似的应用程序。  
可以通过使用该协议读取目标服务器端口上运行的服务版本信息。  
```HTML
dict://<server-ip>:<port>  
```
例如：可以使用该协议进行未授权redis的攻击。  

### 4.1.3 gopher 协议  
> [!note]
> Gopher 协议是一种用于访问和操作互联网资源的协议，类似于 HTTP 协议。很古老，已经为 HTTP 让步。  
> 需要注意的是，Gopher 协议在一些现代浏览器中已被禁用，但在某些特定场景下仍可用，例如一些内部网络应用程序。因此，利用 Gopher 协议进行 SSRF 漏洞利用时需要根据具体情况进行选择。  
> 该协议的默认端口为：**70**

```HTML
gopher://<server:port>/<gopher-path>_TCP-stream  
```
**gopher 协议的利用过程及各步骤原因**：  
gopher 协议需要传输的内容是二进制数据流，而 URL 编码可以将二进制数据流转换成可读的 ASCII 字符，以便在 URL 中传输。所以整体需要进行**两次**的 url 编码。  
1. 第一次 URL 编码：对 payload 进行 URL 编码，这样才能在 URL 中传输。得到 `1st-step-payload`  
2. 由于 gopher 协议中换行符的编码为 `%0A`，而在 HTTP 请求中换行符的编码为 `%0D%0A`，所以在进行第一次 URL 编码时需要将 `%0A` 替换为 `%0D%0A`，以防止 HTTP 请求被截断。得到 `2nd-step-payload`  
3. 第二次 URL 编码：构造 payload 为 `gopher://<server-ip>:<port>/_2nd-step-payload`，并对其整体进行 URL 编码。URL 编码的目的是将特殊字符转义为可用于 URL 的形式，而 `gopher://` 本身也包含特殊字符。因此，将整个 Gopher 协议进行编码可以确保它在传输过程中不会被修改或解释。最终得到了 `final-payload`  
4. 尝试使用`final-payload`进行利用。  

**【未完成】记录一次 gopher 协议的使用**： [技能树-Web/SSRF/POST请求](https://flowus.cn/2c298fab-9ad2-4dfd-932f-d353dfc551d5)  

## 4.2 绕过  
### 4.2.1 `@`符号绕过  
后端程序可能会对访问的 URL 进行解析，对解析出的 host 地址进行过滤，这可能会出现对 URL 参数解析不当，导致可以绕过。  
例如：`http://www.baidu.com@xx.xx.xx.xx`，可以加载`xx.xx.xx.xx`的页面。此类方法在URL跳转漏洞中可以尝试。  

### 4.2.2 **点分割符号替换**  
钓鱼邮件经常使用此方法进行绕过。  
在浏览器中可以使用不同的分割符号，例如：`。`或`、`等来代替`.`。例如：`http://www。qq。com`  

### 4.2.3 IP 转换进制  
可以将 IP 转化为一个16进制、10进制或8进制数字尝试进行绕过。  
`0.0.0.0` 这个 IP 可以直接访问到本地，存在被正则遗忘的可能。

### 4.2.4 使用解析到内网的域名  
如果服务端没有先解析 IP 地址再进行过滤，可以尝试使用 `localhost` 或者 `sslip.io`， `nip.io`，来进行绕过。  
例如：想要访问 `192.168.0.100`，可以尝试使用 `192.168.0.100.sslip.io` 或 `192.168.0.100. nip.io`   
`xip.io` 已经停止服务。  
`sslip.io` 使用方法：[Welcome to sslip.io](https://sslip.io/)  
`nip.io` 使用方法：[nip.io - wildcard DNS for any IP Address](https://nip.io/)  
### 4.2.5 Enclosed alphanumerics 绕过  
[\[封闭式字母数字\]区段所有字符 U+2460-U+24FF | Unicode 符号库 ✏️](https://www.fuhaoku.net/block/Enclosed_Alphanum)  

### 4.2.6 利用跳转  
当后端服务器接收并解析了正确的 URL 而且进行了过滤。可以借用其他网站来进行跳转绕过。例如： [httpbin.org](http://httpbin.org/)  
`http://httpbin.org/redirect-to?url=http://192.168.0.1`，使用上述地址，可以跳转到 `192.168.0.1`  

# 5. 漏洞修复  
---
1. 验证用户输入，确保用户输入的数据包括 url，都经过验证。确保不存在恶意利用的输入内容。
2. 统一报错信息，不提示具体的报错原因。或者不显示报错。
3. 限制请求协议，禁用不常用或者服务端用不到的协议，例如：file，gopher 等。
4. 使用白名单：限制允许访问的 IP 范围，只允许访问特定的域名或 IP。