---
title: JSON Web Token
tags:
  - 计算机网络
creation date: 2024-03-22
done:
---
# 1. JSON Web Token  
---
>[!note]
>**Json Web Token** (JWT)，是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（RFC 7519）。  
>该 token 被设计为紧凑且安全的，特别适用于分布式站点的单点登录（SSO）场景，是目前最流行的跨域认证解决方案。  
>JWT 的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该 token 也可直接被用于认证，也可被加密。  
>JWT 可以使用 HMAC 算法、RSA、ECDSA 等进行签名。   

## 1.1 使用场景  
- **授权**：这是使用 JWT 最常见的场景。一旦用户登录，每个后续请求都将包含 JWT，允许用户访问该令牌允许的路由、服务和资源。单点登录是当今广泛使用 JWT 的一项功能，因为它开销小，并且能够轻松地跨不同域使用。  
- **信息交换**：JSON 网络令牌是各方之间安全传输信息的好方法，因为 JWT 可以签名。  

# 2. JWT 的结构   
---
实际情况的 JWT 格式如下：   
![JWT的格式](https://image.yiscook.top/blog-image/202403221128679.png)  
可以分为三个部分，每个部分之间使用 `.` 进行分隔。**并且在 JWT 中内部没有换行**。   
- Header（头部）  
- Payload（负载）  
- Signature（签名）  
```Plain Text  
Header.Payload.Signature  
```  

## 2.1 Header  
Header 通常由两部分组成：   
- `typ`：token 的类型。JWT 写为 JWT。  
- `alg`：使用的签名算法，如常用的 HMAC、SHA256、RSA（可选项比较多）  
```Plain Text  
{  
  "alg": "HS256",  
  "typ": "JWT"  
}  
```  
这个 JSON 被**Base64Url**编码以形成 JWT 的第一部分。  

## 2.2 Payload  
可以包含声明。声明是关于实体（通常是用户）和附加数据的声明。声明有三种类型：`registered` 注册声明、`public` 公共声明和 `private` 私有声明。  
- 注册声明：不是强制性的，而是推荐的  
    - `iss (issuer)`：签发人  
    - `exp (expiration time)`：过期时间  
    - `sub (subject)`：主题  
    - `aud (audience)`：受众  
    - `nbf (Not Before)`：生效时间  
    - `iat (Issued At)`：签发时间  
    - `jti (JWT ID)`：编号  
- 公共声明：可以由使用 JWT 的人随意定义。但是为了避免冲突，它们应该在 [IANAJSON Web Token注册表](https://www.iana.org/assignments/jwt/jwt.xhtml)中定义，或者定义为包含抗冲突命名空间的 URI。  
```Plain Text  
{  
  "sub": "1234567890",  
  "name": "John Doe",  
  "admin": true  
}  
```  
这个 JSON 被 **Base64Url** 编码以形成 JWT 的第二部分。    
JWT 默认是不会对 Payload 加密的，也就意味着任何人都可以读到这部分 JSON 的内容，所以不要将私密的信息放在这个部分。  

## 2.3 Signature  
Signature 部分是对前两部分的签名，防止数据篡改。  
需要指定一个私钥（secret）。这个密钥只有服务器才知道，不可泄露。然后，使用 `Header` 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。  
```Plain Text  
HMACSHA256(  
  base64UrlEncode(header) + "." +  
  base64UrlEncode(payload),  
  secret)  
```  
签名用于验证消息在此过程中没有更改，并且在使用私钥签名的令牌的情况下，它还可以验证 JWT 的发送者是否是它所说的发送者。  
上述内容可以使用官方提供的 [debugger网站](https://jwt.io/#debugger-io)进行测试。  

![JWT加密前后内容展示](https://image.yiscook.top/blog-image/202403221131493.png)   

# 3. JWT 如何工作   
---
1. 在身份验证中，当用户使用其凭据成功登录时，将返回 JSON Web Token。因 JWT 属于凭据，要小心安全问题，不该保留令牌超过要求的时间。   
2. 每当用户想要访问受保护的路由或资源时，用户代理都应该发送 JWT，通常在使用 HTTP 的请求头中。  
    ```Plain Text  
    Authorization: Bearer <token>  
    ```  
    - JWT 要注意大小，一些服务器不接受超过8 KB 的表头。如需要添加过多信息，可尝试 [Fine Grained Authorization (FGA) at scale for developers - Auth0](https://auth0.com/fine-grained-authorization)  
    - 如果 JWT 在 `Authorization` 标头中发送，跨域资源共享（CORS）不会成为问题，因为它不使用 cookie。   
1. 服务器将检查 `Authorization` 标头中的有效 JWT，如果存在，用户将被允许访问受保护的资源。   

## **3.2 使用 JWT 来访问 API 或者资源：**   
![JWT访问资源流程](https://image.yiscook.top/blog-image/202403221140299.png)  
1. 应用程序或客户端向服务器请求授权。  
2. 服务器通过授权，向应用程序或客户端返回 token。  
3. 应用程序或客户端使用 token 访问受保护的资源或 API。  

# 4. JWT_Tool  
---
[GitHub - ticarpi/jwt\_tool: :snake: A toolkit for testing, tweaking and cracking JSON Web Tokens](https://github.com/ticarpi/jwt_tool)  
[demo.sjoerdlangkemper.nl/jwtdemo/hs256.php](https://demo.sjoerdlangkemper.nl/jwtdemo/hs256.php)  
## 4.1 基础使用方法：  
```PowerShell  
python jwt_tool <JWT>  
# 例如：  
# python jwt_tool.py eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczpcL1wvZGVtby5zam9lcmRsYW5na2VtcGVyLm5sXC8iLCJpYXQiOjE2ODEyNzM2NzcsImV4cCI6MTY4MTI3NDg3NywiZGF0YSI6eyJoZWxsbyI6IndvcmxkIn19.oW0RqRmElN-DveBjsFNgJw1A-cQv4pFjICzWW3PdM34  
```  

## 4.2 其他参数的使用  
- `-T`：`--tamper`，可用于篡改 JWT 的内容  
    ```PowerShell  
    python jwt_tool <jwt> -T  
    ```  
    ![篡改JWT的内容](https://image.yiscook.top/blog-image/202403221143291.png)  
    提示签名没有修改，没有使用签名的模块。所以发送该token到服务器会报错。  
- `-d`：`--dict`，用于破解的字典文件。  
- `-C`：`--crack`，破解HMAC-SHAtoken的secret key。  
    ```PowerShell  
    python jwt_tool <jwt> -C -d <dict-path>  
    ```  
    ![使用字典爆破secret key](https://image.yiscook.top/blog-image/202403221147873.png)    
- `-S`：`--sign`，选择签名使用的算法。  
    - hs256/hs384/hs512 → HMAC-SHA 签名（用-k/-p 指定 secret）   
    - rs256/rs384/hs512 → RSA 签名（使用-pr 指定 RSA 私钥）   
    - es256/es384/es512 → 椭圆曲线签名（使用-pr 指定 EC 私钥）   
    - ps256/ps384/ps512 → PSS-RSA 签名（使用-pr 指定 RSA 私钥）   
    ```PowerShell  
    python jwt_tool <jwt> -T -S hs256 -p 'secret'  
    ```  
    ![指定secret进行加密](https://image.yiscook.top/blog-image/202403221150457.png)  
    发送后的测试结果如下：  
    ![测试结果](https://image.yiscook.top/blog-image/202403221153420.png)  
其他用法，可以使用`-h`进一步了解学习。  

# 5. JWT 的 base64 URL encode 实现流程  
---
1. 将 JWT 的 Header 和 Payload 通过 Base64 编码转换成字符串形式。  
2. 对于 Base64 编码后的字符串，将字符`+`和`/`分别替换为`-`和`_`。  
3. 将字符串末尾的等号去掉。

# 6. 相关链接  
---
[JSON Web Token Introduction - jwt.io](https://jwt.io/introduction/)  
[JSON Web Tokens - jwt.io](https://jwt.io/#debugger-io)  
[开发工具箱 - JWT 在线解密](https://www.box3.cn/tools/jwt.html)  
[JWT基础知识](https://www.wolai.com/ctfhub/hcFRbVUSwDUD1UTrPJbkob)  
[Hacker Tools: JWT\_Tool - The JSON Web Token Toolkit - Intigriti](https://blog.intigriti.com/2021/07/27/hacker-tools-jwt_tool/)  
