---
title: XSS-跨站脚本攻击
tags:
  - web安全
creation date: 2024-03-16
done: true
---

# 1. 跨站脚本攻击概述  
---
> [!note]
> **跨站脚本攻击** (Cross-site scripting，XSS) 指的是攻击者利用网页开发时留下的漏洞，构造出恶意代码注入到网页中，使得用户在浏览网页的过程中，嵌入恶意代码会被执行，从而达到攻击者的目的。   
> 
> 攻击者可以利用跨站脚本漏洞在用户的浏览器中执行任意的脚本代码，包括盗取用户的 cookie 信息、发起恶意请求、修改网页内容等操作。这些攻击都是在用户不知情的情况下进行的，因此跨站脚本攻击对于网站的安全具有很大的威胁。  
> 
> **漏洞成因**：主要因为 web 应用程序在接受用户输入的内容未做充分的检测和过滤。允许用户在提交的数据中掺入代码，并且在未进行编码地输出到浏览器中，导致恶意代码可以在用户的浏览器中执行。

## 1.1 危害  
- 获取用户信息。例如：浏览器信息、IP 地址、cookie 信息等。  
- 钓鱼。  
- 注入木马或广告等恶意链接。  
- 后台增删改网站数据等操作。  
- xss 蠕虫。例如：微博蠕虫：看过某人微博自动关注；贴吧蠕虫：看过某帖子自动回复。  

## 1.2 攻击类型和利用场景  
### 1.2.1 反射型 xss  
也称作非持久型 xss。攻击者将恶意脚本附加到 url 的参数中，并发送给受害者。服务端未经严格过滤处理，会输出受害者的浏览器中，导致浏览器执行恶意代码。  
- 特点：只执行一次。  
- 常见场景：搜索处。  

### 1.2.2 存储型 xss  
也称作持久型 xss。攻击者将恶意代码嵌入到数据中，在其他用户请求后，服务器从数据库中查询数据（含有恶意代码）并发给用户，用户浏览此类页面就可能受到攻击。   
- 常见场景：评论区留言，个人信息等。  

### 1.2.3 DOM 型 xss  
基于 DOM 的 xss，通过对具体的 DOM 代码进行分析，根据实际情况构造 DOM 节点进行 xss 跨站脚本攻击。  
通常是修改 DOM 节点属性或者修改 DOM 节点的 innerHTML 来实现恶意脚本的注入。攻击者可以通过构造特定的 URL 参数或者提交特定的表单数据来触发 DOM 型 XSS 漏洞。  
例如：存在一个搜索页面，用户输入关键词后会通过 JavaScript 将搜索结果显示在页面上，其中搜索关键词会直接拼接到 HTML 中  
- `http://example.com/search.php` 存在 DOM 型 XSS 的搜索页面。  
- `keyword`，接受输入搜索内容的参数。  
```html
<div id="search-result">  
    搜索结果：<?php echo $_GET['keyword']; ?>  
</div>  
```
攻击者可以构造以下payload：  
```html
http://example.com/search.php?keyword=<script>alert('XSS')</script>  
```
当用户访问该 URL 时，恶意脚本会被执行，弹出一个包含 XSS 文本的警告框。  

### 1.2.4 self xss  
自己输入 xss 脚本，但是仅自己能看到。  

# 2. xss利用  
---
## 2.1 xss平台  
1. XSStrike - [GitHub - s0md3v/XSStrike: Most advanced XSS scanner.](https://github.com/s0md3v/XSStrike)  
2. BeEF - [BeEF - The Browser Exploitation Framework Project](https://beefproject.com/)  
5. XSS platform - [XSS平台 - （支持http/https）XSS Platform](https://xss.pt/xss.php?do=login)  
6. XSS platform - [XSS平台-仅用于xss安全测试专用](https://xssaq.com/login/)  
7. BlueLotus - [GitHub - firesunCN/BlueLotus\_XSSReceiver](https://github.com/firesunCN/BlueLotus_XSSReceiver)  

## 2.2 可以利用的HTML事件  

|**事件**|**触发时机**|
|-|-|
|**onload**|在文档或图像加载完成后触发。|
|**onerror**|在文档或图像加载失败时触发。|
|**onmouseover**|在鼠标移动到一个元素上触发。|
|**onmouseout**|在鼠标移出一个元素时触发。|
|**onclick**|在元素被点击时触发。|
|**onsubmit**|在表单提交时触发。|
|**onfocus**|在元素获得焦点时触发.|
|**onblur**|在元素获失去焦点时触发。|
|**onchange**|在元素的值发生改变时触发。|
|**onkeydown**|在按下键时触发。|
|**onkeyup**|在释放键时触发。|
|**onkeypress**|在按住键时触发。|

## 2.3 应对方法  
- `<>`被过滤：尝试构造一些触发事件。例如：使用点击。`' onclick=alert(1)`。先闭合前面的内容，再通过点击进行弹窗。  
- `alert` 函数被过滤：可以使用 `prompt` 和 `confirm` 进行弹窗；也可以使用 JavaScript 的 `String.fromCharCode()` 函数将 ASCII 码转换为 `alert()` 等函数来构造 payload。  
    - `prompt(message, defaultText)`：`message`参数是要显示在提示框中的字符串消息，可以为空字符串或者省略；`defaultText`参数是一个可选参数，用于在文本输入框中提供默认值。  
    - `confirm(defaultText)`：`defaultText`参数是要显示在提示框中的字符串消息。弹出一个包含“确定”和“取消”按钮的对话框，以向用户确认某个操作是否要执行。常见方法如下：  
        ```JavaScript
        if(confirm("确定要执行此操作吗？")) {  
          // 用户点击了“确定”按钮，执行相应操作  
        } else {  
          // 用户点击了“取消”按钮，取消操作  
        }  
        ```
- `.`被过滤：点可以用 `&#46;` 来表示。  
- `on`被过滤：  
    - 使用 `a` 标签：攻击者可以尝试使用 javascript URI 来绕过过滤器，例如：  
        ```HTML
        <a href="javascript:alert('XSS')">Click me</a>  
        ```
    - 使用表单：攻击者尝试构造表单，在表单中使用动作进行绕过，例如：  
        ```HTML
        <form>  
            <button formaction=javascript&colon;alert(1)>M</button>  
        </form>  
        ```
    - 其他可以执行`JavaScript`的属性：  
        - `formaction` 属性：定义在提交表单时，覆盖 `action` 属性指定的地址的地址。如果存在 `formaction` 属性，则使用 `formaction` 指定的地址而不是 `action` 指定的地址。  
        - `action` 属性：定义在提交表单时，将要接收表单数据的地址。一般用于 `<form>` 元素中。  
        - `href` 属性：定义一个超链接目标的 URL，可以是一个 Web 页面、图片、文件等资源的 URL。常用于 `<a>`、`<link>`、`<area>` 元素中。  
        - `xlink:href` 属性：用于定义指向外部资源的 XLink 链接。XLink 是用于在 XML 文档中创建超链接的 W3C 标准。常用于 SVG 中的 `<a>`、`<image>`、`<use>` 等元素中。  
        - `src` 属性：定义资源的 URL，该 URL 指向外部脚本文件、图像文件或媒体文件。常用于 `<script>`、`<img>`、`<audio>`、`<video>` 等元素中。  
        - `content` 属性：定义一个嵌入式的 Web 文档的内容。常用于 `<object>`、`<iframe>` 等元素中。  
        - `data` 属性：定义一个要嵌入文档的数据（例如：图像、音频、视频等）。常用于 `<object>`、`<embed>` 等元素中。  

# 3. 其他内容  
---
## 3.1 htmlentities()  
这是一个 PHP 语言的函数。主要用于将 HTML 特殊字符转义为对应的 HTML 实体，以避免在输出时被解释为 HTML 代码而产生意外效果或者安全漏洞，可以用于防范 XSS 攻击。  
```PHP
string htmlentities(string $string, int $flags = ENT_COMPAT | ENT_HTML401, string|null $encoding = null, bool $double_encode = true)  
```
- `$string`为要转义的字符串  
- `$flags`为可选参数，用于指定转义方式，可选值如下：  
    - `ENT_COMPAT`：默认方式，仅转义双引号和单引号；  
    - `ENT_QUOTES`：转义双引号和单引号；  
    - `ENT_NOQUOTES`：不转义双引号和单引号；  
    - `ENT_HTML401`：使用 HTML 4.01 实体编码；  
    - `ENT_XML1`：使用 XML 1 实体编码；  
    - `ENT_XHTML`：使用 XHTML 实体编码；  
    - `ENT_HTML5`：使用 HTML 5 实体编码。  
- `$encoding`为可选参数，用于指定输入字符串的编码，默认为使用内部字符编码  
- `$double_encode`为可选参数，用于指定是否允许对已经转义的实体再次进行转义，默认为允许。  

## 3.2 HttpOnly  
`HttpOnly` 是一个标志，可以在设置 `HTTP` 响应头中的 `Set-Cookie` 时使用，以确保浏览器无法通过 JavaScript 访问 cookie。如果将 `HttpOnly` 标志设置为 `true`，则浏览器只会在 `HTTP` 请求头中发送 cookie，并且 JavaScript 无法访问 cookie 的值，从而提高了 Web 应用程序的安全性。  
如果 cookie 没有使用 `HttpOnly` 标志，则恶意脚本可以使用 `document.cookie` 属性访问 cookie 的值。  
需要注意的是，`HttpOnly` 标志不是万能的，它仅仅是防范 XSS 攻击的一种手段。攻击者仍然可以通过其他手段来窃取用户的 cookie。  
