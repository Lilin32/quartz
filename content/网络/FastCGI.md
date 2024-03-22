---
title: FastCGI
tags:
  - 计算机网络
creation date: 2024-03-22
done: true
---
>[!note]
>**FastCGI**（Fast Common Gateway Interface）是一个 Web 服务器与应用程序之间通信的协议，它是 CGI 的改进版。与 CGI 不同的是，FastCGI 在 Web 服务器和应用程序之间建立了一个持久性的连接，而不是每个请求都建立一个新的进程或线程。该协议使用**二进制数据流**进行通信，而不是使用文本格式或 JSON 格式。这样可以减少通信数据的大小，提高通信效率。  
>**FastCGI** 支持多种编程语言和 Web 服务器，包括 Apache、Nginx、IIS 等常见的 Web 服务器，以及 PHP、Python、Ruby 等常见的编程语言。  
>**FastCGI** 的主要优点：高性能、可扩展性和灵活性，它可以大幅度提高 Web 应用程序的性能和并发能力。适用于高负载、高并发的 Web 应用场景。  

# 1. FastCGI的数据格式  
---
```JSON
// head -- 消息头
{  
    version             // 协议版本号，一般为1  
    type                // 消息类型  
    requestIdB1         // requestId，16 位长度  
    requestIdB0  
    contentLengthB1     // 消息体长度，16 位长度  
    contentLengthB0  
    paddingLength       // 填充字节数  
    reserved            // 保留字段  
}

// data -- 消息体
{  
    content    // FastCGI 协议约定的请求体  
}

// padding -- 填充字节
{  
    padding   // 用于填充内容，保证消息总长度为8字节的倍数  
}  
```

`requestIdB1` 和 `requestIdB0` 是两个字节的字段，用于表示请求 **ID**，它们是 `request ID` 的高字节和低字节。  
`contentLengthB1` 和 `contentLengthB0` 也是两个字节的字段，用于表示请求体的**长度**，是 `content length` 的高字节和低字节。  

**type：消息类型所具有的选项**  

| **类型字段**                   | **类型数字** | **含义**                  |
| -------------------------- | -------- | ----------------------- |
| **FCGI_BEGIN_REQUEST**     | 1        | 开始请求类型，表示一个新的请求开始。      |
| **FCGI_ABORT_REQUEST**     | 2        | 终止请求类型，表示请求被终止。         |
| **FCGI_END_REQUEST**       | 3        | 结束请求类型，表示请求处理结束。        |
| **FCGI_PARAMS**            | 4        | 参数类型，表示请求参数的键值对。        |
| **FCGI_STDIN**             | 5        | 标准输入类型，表示请求的输入数据。       |
| **FCGI_STDOUT**            | 6        | 标准输出类型，表示响应的输出数据。       |
| **FCGI_STDERR**            | 7        | 标准错误类型，表示响应的错误信息。       |
| **FCGI_DATA**              | 8        | 数据类型，表示响应的二进制数据。        |
| **FCGI_GET_VALUES**        | 9        | 获取值类型，用于获取服务器的配置信息。     |
| **FCGI_GET_VALUES_RESULT** | 10       | 获取值结果类型，表示获取服务器配置信息的结果。 |
| **FCGI_UNKNOWN_TYPE**      | 11       | 未知类型，表示收到了不支持的类型字段。     |

# 2. FastCGI流程  
---
![fastcgi 流程](https://image.yiscook.top/blog-image/202403221120830.png)  
1. 用户通过浏览器发送 HTTP 请求给**web 服务器**。   
2. **web 服务器**收到 HTTP 请求，将请求转发给对应的**web 应用**。**web 应用**收到请求后，生成对应的响应内容。   
3. **web 应用**将响应内容通过**FastCGI 协议**发送给**FastCGI 进程管理器**。   
4. **进程管理器**根据**FastCGI 协议**解析请求，并转给对应的**FastCGI 进程**。   
5. **FastCGI 进程**链接到**数据库服务器**，发送**数据库**的操作请求（读或写）。   
6. **数据库**执行该请求，生成操作结果。   
7. **数据库服务器**将生成的操作结果发送给**FastCGI 进程**。   
8. **FastCGI 进程**将响应内容通过**FastCGI 协议**发送给进程管理器。   
9. **进程管理器**将响应内容返回给**web 应用**。  
10. **web 应用**将响应内容返回给**web 服务器**，**web 服务器**将响应内容返回给浏览器。  

FastCGI 进程本身不会执行数据库语句，只负责将查询请求发送给数据库服务器。在整个过程中，FastCGI 进程充当了 web 应用和数据库服务器之间的重中间层，将两者连接起来。   

# 3. PHP-FPM  
---
php-fpm（PHP FastCGI Process Manager）是 PHP 的一个 FastCGI 进程管理器，用于处理 PHP 程序的请求。相比于传统的 CGI 方式，使用 FastCGI 可以显著提高 PHP 应用程序的性能和响应速度。  
在一个 web 服务器中，通常会有多个请求同时到达，如果每个请求都要创建一个独立的 PHP 进程，那么系统资源就会被大量占用，而使用 php-fpm 则可以让多个请求共享同一个 PHP 进程池，提高系统资源利用率，并且可以设置进程数、内存限制等参数，防止 PHP 程序执行出现问题导致系统宕机。  
php-fpm 是 PHP 自带的进程管理器，它与 PHP 解释器一起打包安装，可以在 php.ini 配置文件中进行相应的设置。  
  