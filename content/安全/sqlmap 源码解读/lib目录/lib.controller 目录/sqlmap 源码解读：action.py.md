---
title: sqlmap 源码解读：action.py
tags:
  - 渗透测试
  - web安全
  - python
  - SQL注入
  - sqlmap
creation date: 2024-05-17
done:
---
# action 函数  
---
## 1. 确定后端使用的数据库
检测 web 应用后端的数据库系统，并选择合适的数据库连接器。并且会将其保存在全局变量 kb 和 conf 中。  
```python
setHandler()
```
若该函数没有识别出所使用的数据库，会通过 html 错误页面继续进行判断。若最终无法获取到数据库类型，会抛出异常。  

## 2. 