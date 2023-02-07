---
title: "c#操作redis"
date: 2023-02-07T15:52:40+08:00
draft: false
description: "c#操作redis"
tags: ["c#","redis"]
categories: ["c#","redis"]
---

## Redis是什么？

redis是一个开源的，面向键/值对的NOSQL的分布式数据库系统

NOSQL指的是非关系型的数据，简单直白地讲就是在非关系型的数据库中不存在表的概念，而是以键值对的方式，

即一个KEY关联一个值的方式进行存储。

redis是一个纯粹为应用而生的高性能数据库系统，非常适合用于持久储存，适应高并发等业务情景。

顺便提一下，redis是一个单线程的程序

## redis是单线程的程序，为什么会这么快

1.大量的线程导致的线程切换开销

2.不存在非必要的内存浪费（因为redis是即使申请内存的，数据多大申请存储的内存就多大）

3.数据结构多样但只做自己的事情。（这样说有点模糊。。）

## redis能存储的五种数据类型

1.string（字符串）

public ActionResult Index()
        {
            
            //创建一个指向服务器Redis连接
            var Client = new RedisClient("127.0.0.1", 6370);
    
            //将一个集合存储在服务器上，存储的类型为string
            //因为在向服务器存储的过程中Redis会将存储的数据序列化为JSON数据，所以在Redis中存储的数据本质是一个字符串
            var UserList = UserInfoService.LoadEntities(u => u.DelFlag == 1).ToList();
            Client.Set<List<UserInfo>>("UserList", UserList);
    
            //获取一个key中的值，和存储的时候一样，读取的时候会对Redis中的数据反序列化。
            List<UserInfo> List = Client.Get<List<UserInfo>>("UserList");
    
            var temp = from s in List
                       select new
                       {
                           Id=s.ID,
                           Name=s.UName,
                           CreateTime=s.SubTime
                       };
            return Json(temp);
        }

 从代码中可以推断当redis内部进行存取所做的序列化和反序列化步骤必定会造成一定的性能损耗，虽然对redis来说影响微乎其微，

但对于某些特殊业务场景下可能造成更加量级的影响，所以我们可以使用hash来进行无需序列化的存储。（仅仅是一个菜鸡的认知，如果大神有幸读到本篇文章请批评我的无知。。）

2.hash（哈希）

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;"> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> ActionResult Hash()
        {
            Client.SetEntryInHash(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">user</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">UName</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">小黄</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.SetEntryInHash(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">user</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">CreateTime</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">18</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> user = Client.Get&lt;UserInfo&gt;(<span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">user</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">以上操作等同于将一个对象拆分出来将其中属性分别存储
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">列如“UName”作为key他的值为“小黄”，“CreateTime”作为key他的值为“18”两者都有不同的key值
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">但他们同时都拥有父级key “user”
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">这种方式无需序列化直接将数据存在了redis中</span>

            <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> View();
        }</span></pre>


*3.list（包含队列，栈的双向链表） 数据结构*

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;"> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> ActionResult List()
        {
            Queue</span>&lt;UserInfo&gt; UserList = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span> Queue&lt;UserInfo&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">();
            UserInfo user1 </span>= <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> UserInfo();
            UserList.Enqueue(user1);
            UserList.Dequeue();
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">.net中的出队入队</span>
            Client.EnqueueItemOnList(<span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">List</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">user1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> count = Client.GetListCount(<span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">List</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">for</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> i = <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">0</span>; i &lt; count; i++<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">)
            {
                Client.DequeueItemFromList(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">List</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            }
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">redis中的队列
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">当redis中队列入队时会将数据存储到redis当中，当redis部署到另一台服务器上时就能实现分布式缓存。<br/><br/></span>
            <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> View();
        }</span></pre>


 

Client.PushItemToList("Item", "111");
///redis中的栈操作，和队列操作无异

* 后面会做一个分布式缓存的列子。
4.set（无序列表）*

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;"> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> ActionResult Set()
        {
            Client.AddItemToSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">1111</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">2222</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            HashSet</span>&lt;<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">string</span>&gt; hashset= Client.GetAllItemsFromSet(<span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">foreach</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> item <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">in</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> hashset)
            {
                </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">输出</span>
<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">            }
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">hashset是无序的，这是.net对hashset的简单操作。</span>
            Client.AddItemToSet(<span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set2</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">1111</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set2</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">2222</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set3</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">1111</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set3</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">2222</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            HashSet</span>&lt;<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">string</span>&gt; hashset1 = Client.GetUnionFromSets(<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">string</span>[] { <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set2</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set3</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> });
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">foreach</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> item <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">in</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> hashset1)
            {
                </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">输出set2,set3中所有的元素</span>
<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">            }
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">并集</span>
            HashSet&lt;<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">string</span>&gt; hashset2 = Client.GetIntersectFromSets (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">string</span>[] { <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set2</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set3</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> });
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">foreach</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">var</span> item <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">in</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> hashset1)
            {
                </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">输出set2,set3中同时存在的元素</span>
<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">            }
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">交集</span>
            <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> View();
        }</span></pre>


使用并集和交集能满足的一些业务场景，列如新浪微博中两个用户共同的粉丝。

* 5.zset(有序列表)

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;"> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> ActionResult Zset()
        {
            Client.AddItemToSortedSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSortedSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">2</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            Client.AddItemToSortedSet(</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">set1</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span>, <span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">3</span><span data-mce-style="color: #800000;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 0);">&#34;</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #808080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 128, 128);">///</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">能让一个集合变成有序</span>
        }</pre>


 

## 两种持久化存储方式

1.快照方式
当对redis中的key操作数量到达一定数值时，将数据写磁盘文件上。（默认方式）
当操作达不到一定值时不会在磁盘存储数据，有可能造成数据丢失。
2.AOF
每个一个时间缎就从内存中取出数据写在磁盘上。
性能损耗
两种方式使用配置文件进行配置

## 使用Redis完成分布式队列（文件并发）

1.定义一个异常类来对全局异常进行捕获，将异常信息存储到redis中
```
using ServiceStack.Redis;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Mvc;

namespace Ym.OA.WebApp.Models
{
    public class MyExceptionAttribute:HandleErrorAttribute
    {   
        ///public static Queue<Exception> ExceptionQueue = new Queue<Exception>();
        ///
        public static IRedisClientsManager ClientManager = 
        new PooledRedisClientManager(new string[] { "127.0.0.1:6379" });
        public static IRedisClient Client = ClientManager.GetClient();
        /// <summary>
        /// 捕获异常，将异常放进队列中。
        /// </summary>
        /// <param name="filterContext"></param>
        public override void OnException(ExceptionContext filterContext)
        {
            base.OnException(filterContext);
            Exception ex = filterContext.Exception;
            Client.EnqueueItemOnList("ErrorQueue", ex.ToString());
           /// ExceptionQueue.Enqueue(ex);
            filterContext.HttpContext.Response.Redirect("/Error.html");
        }
    }
}
```

2.在Global文件中重写Application_Start方法，让程序启动时执行拓展的代码，在当中获取redis中存储的错误信息写入日志。
```
using log4net;
using Spring.Web.Mvc;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Web;
using System.Web.Http;
using System.Web.Mvc;
using System.Web.Optimization;
using System.Web.Routing;
using Ym.OA.WebApp.Models;

namespace Ym.OA.WebApp
{
    // 注意: 有关启用 IIS6 或 IIS7 经典模式的说明，
    // 请访问 http://go.microsoft.com/?LinkId=9394801

    public class MvcApplication :SpringMvcApplication
    {
        protected void Application_Start()
        {
            log4net.Config.XmlConfigurator.Configure();//读取log4配置文件
            AreaRegistration.RegisterAllAreas();
            WebApiConfig.Register(GlobalConfiguration.Configuration);
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            RouteConfig.RegisterRoutes(RouteTable.Routes);
            BundleConfig.RegisterBundles(BundleTable.Bundles);
            //开启线程扫描异常，写入日志。
            string path = Server.MapPath("/Log/");
            ThreadPool.QueueUserWorkItem((a) => {
                while (true)
                {
                    ///判读Redis中存储的队列是否有数据
                    if (MyExceptionAttribute.Client.GetListCount("ErrorQueue")>0)
                    {
                        ///获取错误队列中的信息
                        string ex = MyExceptionAttribute.Client.DequeueItemFromList("ErrorQueue").ToString();
                        if (!string.IsNullOrEmpty(ex))
                        {
                            ILog logger = LogManager.GetLogger("ErrorMessage");
                            logger.Error(ex.ToString());
                        }
                        else
                        {
                            Thread.Sleep(3000);
                        }
                    }
                    else
                    {
                        Thread.Sleep(3000);
                    }
                }
            },path);
        }
    }
}
```

这里只是简单地使用Redis进行简单操作，希望对大家有所帮助。
