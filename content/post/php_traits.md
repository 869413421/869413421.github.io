---
title: "PHP中的traits快速入门"
date: 2023-02-02T14:57:24+08:00
draft: false
description: "PHP中的traits快速入门"
tags: ["PHP","面向对象"]
categories: ["PHP","面向对象"]
---

# traits

在学习PHP的过程中，我们经常会翻阅PHP的官方手册。一般理解能力强悍的人多阅读几遍便可轻松理解其中要领，但往往更多的初学者对官方文档中寥寥数语的描述难以理解。作为一个曾有同样困扰的人，我的经验是遇到这种情况的时候，首先使用搜索引擎翻阅他人分享的学习成果，当知其一二有了概念以后随手写下一些文档，方便巩固知识，日后在工作中有需要时再去深入细节。

## traits是什么？

首先我们先对这个知识有一个基本的概念，你可以先将traits理解成类似include用于代码复用的技术，include针对的是一个类或者其他文件，而traits则是一个针对方法结构的技术，我们使用use关键字就可以将结构体引用到当前的class当中。

## 需求

![](https://img2018.cnblogs.com/blog/1191491/201810/1191491-20181003175449468-1281776626.png)

图中一共存在五个类，分别是基类A以及其子类BCD和一个完全独立的E类，我们有两个方法getSum,getSub。我们需要在B，C，E中同时包含这两个方法，但D类中不包含。

这时候，我们第一个想法大都会是

1.在B，C，E中复制同样的代码实现这两个方法。

2.定义一个接口让B,C,E去实现。

在没有traits之前可能我们大部分人正是如此去实现需求，不管哪种方法最终的方式都是复制代码重用。

然而这些方式的弊端是

1.繁复的复制工作造成的代码冗余。

2.不具备灵活性当需要添加新的方法时每个地方都要修改，难以维护。

traits的出现正是为了解决上述问题

## 如何使用traits

使用traits的方式很简单，和我们定义类的方式相像，除了关键字以为其余一致。

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;">&lt;?<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">php

trait myCode {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> getSum(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span>, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span> + <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> getSub(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span>, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span> - <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
    }

}</span></pre>


 当定义好一个结构体后我们只需要在类里面使用use关键字进行调用，根据我们上面的需求我们在B,C,E中分别use myCode这个tratis

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;">&lt;?<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">php

trait myCode {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> getSum(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span>, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span> + <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> getSub(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span>, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n1</span> - <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$n2</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
    }

}

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> A {
    
}

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span> B <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">extends</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> A {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> myCode;
}

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span> C <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">extends</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> A {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> myCode;
}

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span> D <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">extends</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> A {
    
}

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> E {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> myCode;
}

</span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$b</span> = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> B();
</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> &#39;B调用tratis中的方法成功，方法结果为：&#39; . <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$b</span>-&gt;getSum(10, 20) . &#39;&lt;/br&gt;&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
</span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$c</span> = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> C();
</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> &#39;C调用tratis中的方法成功，方法结果为：&#39; . <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$c</span>-&gt;getSum(10, 20) . &#39;&lt;/br&gt;&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
</span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$e</span> = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> E();
</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> &#39;E调用tratis中的方法成功，方法结果为：&#39; . <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$e</span>-&gt;getSum(10, 20) . &#39;&lt;/br&gt;&#39;;</pre>


在代码中我们分在每个类中调用了我们定义的方法结构，从而我们不需要在每个类中对方法进行描述，因为程序已经将tratis中的方法自动添加到了每一个类中，这样我们就见面了各种手动繁复的操作，而如果程序后期需要对这几个类拓展的时候只需要对定义的tratis进行修改就可以达到预设的目的，极大地提交了可维护性。

运行这段代码的返回结果为：

![](https://img2018.cnblogs.com/blog/1191491/201810/1191491-20181003214355437-2060663259.png)

最终我们的程序结构如下

 

![](https://img2018.cnblogs.com/blog/1191491/201810/1191491-20181003214738582-1929485096.png)

这样我们就算是对tratis进行了一个简单入门，但应该已经满足我们日常开发的需求；

如果你需要深入了解更多细节可以参阅一下文章

1.https://blog.csdn.net/qq_16142851/article/details/80437560 

2.https://segmentfault.com/a/1190000008009455
