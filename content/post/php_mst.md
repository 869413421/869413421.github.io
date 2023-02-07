---
title: "PHP（基础面试题）"
date: 2023-02-02T14:57:24+08:00
draft: false
description: "PHP基础面试题"
tags: ["PHP","面试题"]
categories: ["PHP","面试题"]
---

1. 引用变量

(1) 引用变量的概念

引用变量指引用变量是指定义不同名称的对象，他们的指向同一个内存空间，不会开辟新的内存。是基于CopyAndWrite实现的。

(2) PHP变量的机制

PHP的变量是基于CopyAndWrite的机制进行实现的，比如定义了变量a,变量b=a,这个时候变量的的内存空间是一致的，b只是指向a的内存空间。只有修改了a的变量内容时，才会开辟一块新的空间重新存储变量a，而变量b的内存空间不变。

unset只会取消变量的引用，而不会去销毁变量空间。只有等GC进行清理的时候才会销毁占用的空间

Object类型本来就是基于引用实现的，两个对象修改值会彼此影响。需要复制对象时候使用clone

真题：

<?php

 

    $data=["a","b","c"];
    
    //循环过后data是什么？
    
    foreach($data as $key=>$val){
    
        $val=&$data[$key];
    
    }

 


    //data=["b","c","c"]
    
    var_dump($data)

?>

2. 常量以及数据类型

(1) .PHP字符串的定义方式和各自区别

单引号:单引号不能解析变量，单引号不能解析转义字符，只能解析单引号和反斜杠本身，单引号效率更高

双引号：双引号可以解析变量可以解析转义字符，可以使用{}解析变量

Heredoc：类似双引号，用于处理大文本

Newdoc：类似单引号，用于处理大文本

(2) .PHP的数据类型

三大数据类型

标量：

浮点数Float(不能用于比较运算中)

<?php 

    $a=0.1;
    
    $b=0.7;

 


    echo $a+$b==0.8

//输出为false,因为浮点数不能运用于比较运算当中，$a+$b==0.7999999

//因为转换成二进制的判断当中，会有精度丢失

?>

整形

字符串

布尔类型

false的七种情况

0，0.0，’ ’，””，’0’，false，array()，null

复合

数组，对象

特殊

资源

NULL

(3)超全局变量

$GLOBALS

$_SERVER

$_REQUEST

$_POST

$_GET

$_FILES

$_ENV

$_COOKIE

$_SESSION

(3) Null

null的三种情况

直接赋值为NULL,未定义的变量，unset变量

(4) 常量

定义常量difine是函数，const是语言结构。

const可以定义类的常量，而difine不能定义，常量定义后不能修改不能删除

一定义常量

__FILE__

__LINE__

__DIR__

__FUNCTION__

__CLASS__

__TRAIT__

__METHOD__

__NAMESPACE__

真题

用PHP写出当前客户端的IP和服务端的ID？

<?php 

    $_SERVER['SERVER_ADDR'];
    
    $_SERVER['REMOTE_ADDR'];

?>

__FILE__是什么？

当前的文件路径和文件名称

3. 运算符

(1) foo()和@foo()之间的区别？

@是错误控制符，放置在一个PHP表达式之前，表达式产生的错误信息都会被忽略掉

(2) 运算符优先级

递增/递减

!

算术运算符

大小比较

相等比较

引用

位运算符^

位运算|

逻辑与

逻辑或

三目运算符

赋值

and

xor

or

(3) ==和===的区别

==比较值是否相等，===比较值是否相等并且比较类型

(4) 递增/递减不影响布尔值

(5) 递减NULL没有效果

(6) 递增NULL值为1

(7) 递增和递减在前就先运算后返回，反之就是先返回再运算

真题

<?php 

   $a=0;

   $b=0;

 

   if($a=3>0||$b=3>0)

   {

       $a++;
    
       $b++;

   }

 

   echo $a.PHP_EOL;

   echo $b.PHP_EOL;

   //输出结果为11

?>

4. 流程控制

（1）PHP遍历数组的三种方式和各自区别

使用for:只可以遍历索引数组

使用foreach:可以遍历索引数组和关联数组，会使用reset()操作，对数组指针进行重置

使用list(),each(),while循环可以遍历索引数组和关联数组，这种方式不会重置数组的指针

（2）if和elseif

if和elseif组合起来只会被执行一次，如果满足其中一个条件，其他的条件不会再被执行，如果所有条件不满足，会执行else。书写if elseif原则是，把可能较大的书写在前面

（3）switch case

和if不同的是，switch的控制表达式后面的数据类型只能是整形，浮点型或者字符串

Continue语句作用到switch的作用类型于break

跳出多层循环可以是用continue2

Switch 底层会生成一个索引表，当满足条件后会直接跳转的需要执行的代码，相对来说效率较高

真题

PHP种如何优化多个if elseif 的情况

1. 把可能性较大的表达式书写在前面
2. 如果判断的类型是整形或者浮点或者字符串，并且条件比较复杂，可以使用switch
3. 自定义函数以及内部函数考点

(1) 全局变量以及局部变量的理解

(2) 静态变量的理解

① 仅会被初始化一次

② 初始化时需要赋值

③ 执行函数值会被保留，不会马上释放内存

④ static是局部变量

⑤ 真题

<?php 

    //输出程序的输出结果

   $count=5;

   function get_count(){

       static $count;
    
       return $count++;

   }

 

   echo $count;

   ++$count;

   echo get_count();

   echo get_count();

 

   //输出结果为5 null 1

?>

(3) 外部文件导入

① Include include加载未找到文件时结构会发出一条警告，include_once只包含一次

② require require会产生一个致命错误,require_once只包含一次

(4)  系统内置函数

① 日期函数

② IP函数

③ 数组处理函数

6. 文件操作

(1) 文件读取写入

① fopen,用来打开一个文件，打开时指定打开模式。

② 常用函数

1) fread()
2) fgets()读取一行
3) fgetc()读取一个字符
4) fcolse()关闭文件
5) file_get_contents()
6) file_put_contents()
7) file 把文件读取到一个数组里面
8) readfile 读取整个文件并且输出到缓冲区
9) allow_url_fopen,远程读取文件
10) filesize()
11) Cop  y()

③ 目录常用函数

1) basename()
2) dirname()
3) Pathinfo()
4) opendir()
5) readdir()
6) closedir()
7) rewinddir()
8) rmdir()
9) mkdir()

(2) 真题

① 不停地在文件头部写入helloword

<?php 

    $file="./hello.text";
    
    $handle=fopen($file,'r');
    
    $content=fread($handle,filesize($file));
    
    fclose($handle);

 


    $content="hello word".$content;
    
    $handle=fopen($file,'w');
    
    fwrite($handle,$content);
    
    fclose($handle);

?>

② 通过PHP函数对目录进行循环遍历

<?php 

   function loopDir($dir){

        $handle=opendir($dir);

 


        while($file=readdir($handle))
    
        {
    
            echo $file.PHP_EOL;
    
            if($file!='.'&& $file!='..' ){
    
                if(filetype($dir.'/'.$file)=='dir'){
    
                    loopDir($dir.'/'.$file);
    
                }
    
            }
    
        }


​    

   }

 

   loopDir("./a")

?>

7. 会话控制

(1) 为什么使用会话技术

① 因为客户端和服务端基于http协议进行通讯，http是无状态的，所以不能记录两次请求是否属于同一个用户。

(2) Cookie

① 存储在浏览器中，用来记录信息片段的一个文件，文件中包含服务端的信息

1) setcookie
2) $COOKIE

② Cookie的优点

1) 存储在客户端，不占用服务器资源

③ Cookie的缺点

1) 存储在客户端中，可以被篡改，不安全
2) 可以被禁用

(3) Session

①  Session是存储在服务器中的文件，用来记录信息片段的一个文件，Session依赖于cookie。

1) Seesion缺点

a. 存储在服务器，占用服务器资源

2) Session优点

a. 安全性较高

8. 面向对象

(1) 类权限控制符

① Public，可以在类的外部，继承中使用

② Protected 可以在子类的内部使用，单不能外部使用

③ Private 只能在类的内部使用

9. 网络协议

(1) 状态码

① 五大类

1) 1xx，信息类状态码，表示请求正在处理
2) 2xx，成功状态码，正常处理完毕

a. 200成功

b. 204 成功不返回内容

c. 206 成功返回一部分内容

3) 3xx，重定向，表示需要附加额外操作

a. 301 永久重定向

b. 302 临时重定向

c. 303 请求资源需要使用get方法重新请求

d. 304 允许请求资源，但判断后不能返回资源使用304

e. 307 临时重定向

4) 4xx，客户端错误，表示请求服务端无法处理

a. 400 请求报文有误

b. 401 请求需要认证信息

c. 403 服务器拒绝请求

d. 404 找不到资源

5) 5xx，服务端错误，表示服务端错误，无法处理请求

a. 501 执行请求中发生错误

b. 503 服务繁忙

② OSI七层模型

1) 物理层

a. 建立维护断开物理连接

2) 数据链路层

a. 建立逻辑连接，进行硬件地址寻址，差错校验等功能

3) 网络层

a. 进行逻辑地址寻址，实现不同网络之间路径的选择

4) 传输层

a. 定义传输数据的协议端口好，以及流控和差错校验

b. ＴＣＰ

c. ＵＤＰ　

5) 会话层

a. 建立，管理，终止会话。

6) 表示层

a. 数据的表示，安全，压缩

7) 应用层

a. 网络服务与最终用户的一个接口

b. Ｈｔｔｐ

a) 工作特点

i. 基于Ｂ／Ｓ模式

ii. 通信开销小，简单快速，传输成本低

iii. 使用灵活，可以使用超文本传输协议

b) 工作原理

i. 客户端发送请求给服务端，创建一个ＴＣＰ连接，指定端口号，连接到服务器。服务器监听浏览器的请求，解析请求成功后，向客户端返回状态信息以及数据内容

c. ｆｔｐ

d. ｔｆｔｐ

e. ｓｍｔｐ

f. ｓｎｍｐ

g. ｄｎｓ

h. ｔｅｌｎｅｔ

i. ｈｔｔｐｓ

j. Ｄｈｃｐ

③ 常见ＨＴＴＰ协议请求响应头

1) Content-Type　请求实体的类型信息
2) Accept　指定客户端能接受的内容
3) Origin　最初请求来源
4) Cookie　请求携带Ｃｏｏｋｉｅ内容
5) Cache-Control　缓存机制
6) User-Agent　请求的用户信息
7) Referreｒ　请求的连接
8) X-Forwarded-For　客户端的真实ＩＰ
9) Access-Control-Allow-Origin　允许请求的域名
10) Last－Modified　请求最后的响应时间
