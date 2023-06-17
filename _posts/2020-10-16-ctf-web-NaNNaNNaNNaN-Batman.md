---
layout: post
title: Web进阶题-NaNNaNNaNNaN-Batman
excerpt: "攻防世界进阶Web题目-NaNNaNNaNNaN-Batman"
date:   2020-10-16 22:09:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 下载文件web100，由于无文件后缀名，故使用Hex Editor打开目标文件进行分析

   * 大体可以看出文件内容为JavaScript代码
   * 由于文件一开始的内容为"\<script\>"，由于JavaScript文件中是不会出现标签的，所以该文件为HTML文件，JavaScript通过script标签内嵌在HTML文档中

2. 将文件名更改为web100.html，使用浏览器打开，进行进一步分析

   * 网页内容为一个输入框和一个按钮

   * 虽然有大量不可读代码，通过Chrome的JavaScript代码高亮可以看出，JavaScript代码中先定义了一个名为'\_'的字符串，之后执行了一个for循环对'\_'代码进行了处理，之后使用eval将'\_'字符串作为JavaScript代码进行了执行

3. 使用Hex Editor的Insert Mode对web100.js进行编辑，将eval函数修改为console.log，使用浏览器打开，从F12的console中获取真正执行的JavaScript代码

   ```javascript
   function $() {
       var e = document.getElementById("c").value;
       if (e.length == 16)
           if (e.match(/^be0f23/) != null)
               if (e.match(/233ac/) != null)
                   if (e.match(/e98aa$/) != null)
                       if (e.match(/c7be9/) != null) {
                           var t = ["fl", "s_a", "i", "e}"];
                           var n = ["a", "_h0l", "n"];
                           var r = ["g{", "e", "_0"];
                           var i = ["it'", "_", "n"];
                           var s = [t, n, r, i];
                           for (var o = 0; o < 13; ++o) {
                               document.write(s[o % 4][0]);
                               s[o % 4].splice(0, 1)
                           }
                       }
   }
   document.write('<input id="c"><button onclick=$()>Ok</button>');
   delete _
   ```

4. 可以看出进行了四个正则匹配，如果匹配成功，则从四个数组轮流读取首元素中拼接出真正的flag并写到document中

   * 字符串长度为16
   * 字符串以be0f23开头
   * 字符串中有233ac
   * 字符串以e98aa结尾
   * 字符串中有c7be9

   ```txt
   Result:be0f233ac7be98aa
   ```

5. 所以当输入框内容为"be0f233ac7be98aa",网页将打印flag

6. 发现flag，flag{it's_a_h0le_in_0ne}

7. 提交，答案正确


## 独立思考

### 1. 本题中的JavaScript代码有哪些特殊的？

* eval(string)，是全局函数，将字符串当作脚本执行

* with，是关键字，使用with语句关联对象，在with代码块内每个变量首先被认为是一个局部变量，如果局部变量和关联对象的某个属性同名，则这个局部变量会指向关联对象的属性

  * ```javascript
    var obj ={a:1,b:2,c:3}
    ```

  * 一般的修改方式：

    ```javascript
    obj.a=233;
    obj.b=666;
    obj.c=888;
    ```

  * 通过with的修改方式：

    ```javascript
    with(obj){
        a=233;
        b=666;
        c=888;
    }
    ```

* pop()，不是处理栈的函数，而是处理数组的函数，该方法将删除array的最后一个元素，并将该元素返回，如果数组为空，则返回undefined。

* join(separater)，处理数组的函数，将所有数组的元素以separater为分隔符进行拼接成一个字符串

* splice(index,howmany,new......)，处理数组的函数，删除index位置的howmany个元素，第三个及以后的参数是可选的，用于指定替换删除掉的元素的元素

* split(separater，howmany)，处理字符串的函数，是join()的逆过程，用于将一个字符串分割为字符串数组

### 2. 本题中新学会了哪些正则语法？

* \[^ABC\]，匹配除了\[\]中字符的字符
* ^，匹配字符串的开始位置
* $，匹配字符串的结束位置

### 3. 本题中JavaScript字符串经过了哪些处理被解密出来的？

```javascript
for(Y in $='....')
    with(_.split($[Y]))
        _=join(pop())
```

* for/in循环会遍历对象的属性

  ```javascript
  var person={name:"mon",age:233}
  var x;
  for(x in person){
      document.write(person[x]);
  }
  ```

  通过使用浏览器调试发现，for/in会把字符串当作char[]，Y是数组索引号，$[Y]

* 每次循环中

  * 会把"\_"字符串使用$[Y]分割为数组
  * 接下来是与with关键字相关的极具迷惑的一段代码
    * 对"\_"分割出来的数组执行pop()，此时删除了最后一个元素，并返回了一个最后一个元素
    * join()将pop()出来的值作为参数，即separater分隔符，对"_"字符串的"分割后且删除最后一个元素的"数组进行拼接为字符串，填充就是删除前的最后一个元素
    * 将操作后的字符串赋值给"\_"参数

以第一个循环为例，$变量为13B的字符串

* Y的值为0
* $[Y]为0x0f
* 对"\_"字符串以0x0f进行分割为数组
* pop()出来的最后一个元素为"ment"
* join("ment")，将ment插入到数组的元素之间
* 完成了对"\_"字符串的一次解密

以此类推，一共进行13次循环，完成对"\_"的解密，重新恢复出真正的字符串


## 产生过的疑问

1. 本题中的JavaScript代码有哪些特殊的？
2. 本题中新学会了哪些正则语法？
3. 本题中JavaScript字符串经过了哪些处理被解密出来的？

