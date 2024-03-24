---
title: MyBatis的SQL注入
tags:
  - web安全
  - SQL注入
  - Java
creation date: 2024-03-24
done:
---
# 1. 编译和预编译  
---
`Statement` 和 `PreparedStatement` 是 JDBC API 中用于执行 SQL 查询的两种接口，它们之间的主要区别在于功能和性能：   

|              | `Statement`                                                               | `PreparedStatement`                                                              |
| ------------ | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **用途**       | 用于执行基于**字符串**的 SQL 查询，适用于静态 SQL 语句，如 DDL（数据定义语言）语句（CREATE, ALTER, DROP 等） | 用于执行**参数化**的 SQL 查询，适用于需要多次执行的 SQL 语句，如 DML（数据操作语言）语句（INSERT, UPDATE, DELETE 等）。 |
| **参数传递**     | 不能在运行时接受参数，因此不适用于需要动态参数的查询                                                | 可以在运行时接受参数，支持动态 SQL 查询，提高了代码的可读性和维护性。                                            |
| **性能**       | 性能较低，因为每次执行SQL查询时都需要重新编译和解析SQL语句                                          | 性能较好，使用预编译 SQL 语句，减少了数据库的编译和解析开销。特别是在执行相同 SQL 语句多次时，性能优势更明显。                     |
| **数据类型处理**   | 不支持                                                                       | 支持插入异常数据类型，如 `Timestamp`、`InputStream` 等。                                        |
| **批量操作**     | 不支持                                                                       | 可以与批量操作一起使用，允许一次性执行多个 SQL 语句，提高了执行效率。                                            |
| **SQL 注入防护** | 不支持                                                                       | 通过预编译 SQL 语句和参数化查询，有效防止了 SQL 注入攻击。它会对参数值进行转义，确保恶意代码不会被执行。                        |
# 2. MyBatis 的 SQL 注入  
---
MyBatis 的 SQL 语句可以编写在**注解**和 **xml 文件**中，并且 MyBatis 支持使用 generator 自动生成 SQL 语句。  
MyBatis 支持使用两种传入参数的方式，分别是 `#{}` 和 `${}`  
## 2.1 传入参数方式  
- `#{}`：使用预编译的方式生成 SQL 语句，天然具有预防 SQL 注入的优势。
- `${}`：使用字符串拼接的方式生成 SQL 语句，不支持预防 SQL 注入的。

## 2.2 模糊查询  
例如：要查询重名学生的信息  
```SQL
Select * from student where name like ‘%#{name}%’;
```
这种情况下，使用 `#{}` 会产生报错，可能会直接修改为如下内容：`Select * from student where name like ‘%${name}%’;`，如此会产生 SQL 注入。  
正确写法：  
```SQL
select * from student where name like concat(‘%’,#{name}, ‘%’);
```

## 2.3 in 关键字  
例如：查询多个 id 的学生信息  
```SQL
Select * from student where id in (#{ids});
```
这种写法下会产生报错，但是如果提前使用字符串拼接，将 id 进行拼接后得到 ids，再使用 `${}` 进行参数传输，同样会产生 SQL 注入。  
正确写法是需要使用到 MyBatis 的 foreach  
```SQL
Select * from student where id in 
<foreach collection="ids" item="item" open="("separatosr="," close=")">
	#{ids} 
</foreach>
```

## 2.4 order by 关键字  
这种情况下应该是在 Java 代码部分进行安全编码，设置一个字段/表名数组，仅允许用户传入索引值。这样保证传入的字段或者表名都在白名单里面。 

# 3. MyBatis Plus 安全编码  
---
- 使用 `$` 占位符是存在安全风险的，在编码过程中要尽可能使用 `#` 占位符。  
- 使用框架提供的 CRUD 接口及 `QueryWrapper`(`LambdaQueryWrapper`) 和 `UpdateWrapper`(`LambdaUpdateWrapper`)，来生成 SQL 语句。  
- 在使用 xml 文档编写 SQL 语句，可使用 MyBatis 提供的动态 SQL，减少使用 SQL 字符串的拼接。  
	- [动态 SQL\_MyBatis中文网](https://mybatis.net.cn/dynamic-sql.html)  
- 可以使用 SQL 注入工具类：`SqlInjectionUtils`，进而防止 SQL 注入。有两个正则表达式的变量：
	- `SQL_SYNTAX_PATTERN`：检测 SQL 语句中常见的 SQL 关键字、操作符和其他可能与注入攻击相关的关键字  
	- `SQL_COMMENT_PATTERN`：检测SQL语句中的注释