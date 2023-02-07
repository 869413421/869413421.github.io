---
title: "laravel中使用自定义的Common类"
date: 2023-02-07T15:52:40+08:00
draft: false
description: "laravel定义common"
tags: ["PHP","laravel"]
categories: ["PHP","laravel"]
---

​	众所周知，laravel是一款高度集成的开发框架，框架内置非常多的操作方法，从而保证了我们的开发效率。但是在日常中为了满足我们的个性化业务，也需要自己去编写工具类，在laravel中我们完成编写后还需要重新去对compoer的自动加载类进行重新加载。

​	首先在主要代码目录app下创建一个test.php1

![](https://img2018.cnblogs.com/blog/1191491/201903/1191491-20190325172512744-974044526.png)

　　然后还需要在根目录的composer.json中的autoload的file数组中注册我们刚才的helper类

![](https://img2018.cnblogs.com/blog/1191491/201903/1191491-20190325173122233-1191128955.png)

　　最后在项目根目录下执行compoer  dump-autoload命令，这样我们的类会被compoer自动加载了，在项目中直接描述我们的方法名就可以正常使用了。
