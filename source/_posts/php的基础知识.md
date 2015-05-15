title: "php的基础知识"
date: 2015-05-15 21:35:55
tags: 学习笔记
---
# PHP 学习指南

`php学习`                                  `2014 11 29`                                         `homker`  

* PHP的使用环境
* PHP的基本语法
* PHP链接数据库
* PHP和HTTP协议
* PHP和文件的读写
* PHP和oop


## PHP的使用环境

* php 是一门脚本语言通常用来处理页面端传输过来的数据，并将数据和数据库进行交互。
* php 是一门运行在虚拟机上的语言，这个特点和java的机制是一样的。
* php 是一门运行在服务器上的语言，当然，这个特点在php5.4之后被削弱了，php可以通过`Artisan` 来运行属于自己的控制台程序，但是大多数情况下，我们还是会使用`apache`或者`nginx`来运行php的程序，这样能够减少我们很多工作量，同时提高我们的工作效率。
* 在`linux`环境下，如果是`ubuntu`或者是`debian`可以直接使用

```shell
 sudo apt-get install php5 mysql-server mysql-client nginx
```
* 在`window`环境下，你可以选择安装`wamp`软件来集成化安装php apache mysql phpmyadmin 环境。
<!--more-->
## PHP的基本语法

* php 是一门脚本语言，它是基于C的CGI的脚本语言。
* php 语法和c的语法是一致的，请看以下的几个范例

```php
  $a  =  1;//定义变量$a 并给它赋值
  $b = $a + 1;//基于变量的传递的同时给变量赋值
  $c = $a + "string";//error 不同类型的变量不能直接相加，虽然 php 不会报错
  $d = $a.'' . "string";//字串的拼接
  
  //array
  
  //c int a[num];
  $array = array('key'=>'value','key2'=>'value2');
  $value2 = "value2";
  $value2 == $array['key2'] ? true : false;//true
  $array[] = $a;
  
  for($i=0;$i<num;$i++){
     //循环体 
  }
 
  foreach($array as $key => $value){
    echo "<li><a href='".$value."'>".$key."</a></li>";
      //循环体
  }
  
  if(true){}
  else{}
  elseif(){}
  Conditions ? true : false;

  switch(){
      case  : ; break;
      default:;
  }
```

## PHP 链接数据库

* 我们在这里使用的是`mysql`数据库，当然，你们可以选用其他的数据库，使用方式，我就不一一描述了。
* 从`php5.4`之后数据链接都是使用`OPD`的链接方式。

```php
    //传统的数据库链接方式
    mysql_connect('database_address:port','username','password','newlink','linkflag');
    mysql_select_db('databases_name');//选择数据库
    $rs = mysql_query('sql语句');//执行sql语句并返回执行句柄
    $result = mysql_fetch_array($rs);//将mysql的数据句柄转换成数组
    mysql_fetch_row($rs);
    mysql_fetch_assoc($rs);
    mysql_fetch_object($rs);
    
    //现在的数据库链接方式
    
    $mysqli = new mysqli();
    //$mysqli::connect();
    $mysqli->connect('database_address:port','username','password',"databases_name");
    $rs = $mysqli->query('sql语句');
    $result = mysqli->fetch_array($rs)
    
```

## PHP 和 http 协议

* URI 路由处理

```
   <scheme>://<host>:<port>/URI ? query=data
    http   :// class.ecjtu.net : 80 /api.php ? classID = 201221100902
```
* `GET` `POST` 的两种方法

```php
    $_GET['name']//直接用超全局变量来访问
    $_POST['name']
```
* `cookie` `session` 的使用
    * `cookie` 是一种存放在浏览器上的数据包。通过`url`（`http`）来进行存储。
    * `session` 则是存放在服务器上的。每一个`php-fpm`在服务器上开辟一个属于他的`session对话`，通过`PHPSESSIONID`来维护这个`session`的链接。
    * 当浏览器关闭的时候，`session`的数据会被清理，除非保留`PHPSESSIONID`。`cookie`的会保留。
    
```php
    $_COOKIE[]//以数组的方式存放的是所有的cookie
    set_cookie(cookie_name,cookie_value);
    session_start();
    $_SESSION[]//以数组的形式来存放所有的session
    $_SESSION['name'] = $value;
```
## PHP文件处理
* php `file_get_content()`函数。

```php
$path = 'your_file_path';
 if(file_exist($path)){
     $file_value = file_get_content($path);
     $file_value = fopen($path);
 }
```


