---
title: "laravel+Redis简单实现队列通过压力测试的高并发处理"
date: 2023-02-02T14:57:24+08:00
draft: false
description: "laravel+Redis简单实现队列通过压力测试的高并发处理"
tags: ["laravel","redis"]
categories: ["laravel","redis"]
---

### 秒杀活动

在一般的网络商城中我们会经常接触到一些高并发的业务状况，例如我们常见的秒杀抢购等活动，

在这些业务中我们经常需要处理一些关于请求信息过滤以及商品库存的问题。

在请求中比较常见的状况是同一用户发出多次请求或者包含恶意的攻击，以及一些订单的复购等情况。

而在库存方面则需要考虑超卖这种状况。

下面我们来模拟一个简单可用的并发处理。

 

直接上代码

### 代码的流程

1.模拟用户请求，将用户写入redis队列中

2.从用户中取出一个请求信息进行处理（可以在这个步骤做更多的处理，请求过滤，订单复购等）

3.用户下单（支付等），减少库存。下面使用了两种方式进行了处理，一种使用了Redis中单线程原子操作的特性使程序一直线性操作从而保持了数据的一致。

另外一种是用了事务进行操作，可以根据业务进行调整，这里就不一一描述了。

 

实际的业务状况更为复杂，但更多的是出于对基础思路的拓展。

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px;">&lt;?<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">php

namespace App\Http\Controllers\SecKill;

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> App\Http\Controllers\Controller;
</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> Illuminate\Support\Facades\DB;
</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">use</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> Illuminate\Support\Facades\Redis;

</span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">class</span> SecKillControllers <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">extends</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> Controller {

    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> SecKillTest() {<br/>　　　　　///在此之前我们已经将一千过用户写入了redis中了
        </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span> = Redis::lpop(&#39;user_list&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);<br/>　　　　　///取出一个用户<br/>　　　　　///<br/>　　　　　///一些对请求的处理<br/>　　　　　///
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (!<span data-mce-style="color: #008080;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 128);">is_null</span>(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">)) {<br/>　　　　　　　///将需要秒杀的商品放入队列中
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;AddGoodToRedis(1<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);<br/>　　　　　　　///需要注意的是我们如果写的是秒杀活动的话，需要做进一步的处理，例如设置商品队列的缓存等方式，这里就实现了<br/><br/>　　　　　　　///下订单减库存
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;GetGood(1,<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        }
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> DoLog(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$log</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #008080;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 128);">file_put_contents</span>(&#34;test.txt&#34;, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$log</span> . &#39;\r\n&#39;,<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> FILE_APPEND);
    }
    
    </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/*</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">*
     * 重点在于Redis中存储数据的单线程的原子性，！！！无论多少请求同时执行这个方法，依然是依次执行的！！！！！
     * 这种方式性能较高，并且确保了对数据库的单一操作，但容错率极低，一旦出现未可预知的错误会导致数据混乱；
     </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">*/</span>
    <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> GetGood(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span>,<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span> = \App\Goods::find(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (<span data-mce-style="color: #008080;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 128);">is_null</span>(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">)) {
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;商品不存在&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> &#39;error&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        }
    
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/去除一个库存</span>
        <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span> = Redis::lpop(&#39;good_list&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/判断取出库存是否成功</span>
        <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (!<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;取出库存失败&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> &#39;error&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        } </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">else</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> {
            </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/创建订单</span>
            <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span> = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> \App\Order();
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;good_id = <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">good_id;
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;user_id = <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">save();
    
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$ok</span> = DB::table(&#39;Goods&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">)
                    </span>-&gt;where(&#39;good_id&#39;, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">good_id)
                    </span>-&gt;decrement(&#39;good_left&#39;, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (!<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$ok</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
                </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;库存减少失败&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
                </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
            }
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> &#39;下单成功&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        }
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> AddUserToRedis() {
        </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_count</span> = 1000<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">for</span> (<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span> = 0; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span> &lt; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_count</span>; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span>++<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">try</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> {
                Redis</span>::lpush(&#39;user_list&#39;, <span data-mce-style="color: #008080;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 128);">rand</span>(1, 10000<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">));
            } </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">catch</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">Exception</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$e</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
                </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$e</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">getMessage();
            }
        }
        </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_num</span> = Redis::llen(&#39;user_list&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        dd(</span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$user_num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> AddGoodToRedis(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
    
        </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span> = \App\Goods::find(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span> == <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">null</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;商品不存在&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        }
    
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/获取当前redis中的库存。</span>
        <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$left</span> = Redis::llen(&#39;good_list&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/获取到当前实际存在的库存，库存减去Redis中剩余的数量。</span>
        <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$count</span> = <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;good_left - <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$left</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">        dd($good-&gt;good_left);</span>
        <span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/将实际库存添加到Redis中</span>
        <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">for</span> (<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span> = 0; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span> &lt; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$count</span>; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$i</span>++<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
            Redis</span>::lpush(&#39;good_list&#39;, 1<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
        }
        </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span> Redis::llen(&#39;good_list&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
    }
    
    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">public</span> <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">function</span> getGood4Mysql(<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
        DB</span>::<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">beginTransaction();
        </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/开启事务对库存以及下单进行处理</span>
        <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">try</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> {
            </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">/创建订单</span>
            <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span> = <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">new</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> \App\Order();
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;good_id = <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">good_id;
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;user_id = <span data-mce-style="color: #008080;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 128);">rand</span>(1, 1000<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$order</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">save();
    
            </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span> = DB::table(&#34;goods&#34;)-&gt;where([&#39;goods_id&#39; =&gt; <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$id</span>])-&gt;sharedLock()-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">first();
            </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">对商品表进行加锁(悲观锁)</span>
            <span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">good_left) {
                </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$ok</span> = DB::table(&#39;Goods&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">)
                        </span>-&gt;where(&#39;good_id&#39;, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$good</span>-&gt;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">good_id)
                        </span>-&gt;decrement(&#39;good_left&#39;, <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$num</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
                </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">if</span> (<span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$ok</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
                    </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);"> 提交事务</span>
                    DB::<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">commit();
                    </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">echo</span>&#39;下单成功&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
                } </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">else</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> {
                    </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;库存减少失败&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
                }
            } </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">else</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);"> {
                </span><span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$this</span>-&gt;DoLog(&#34;库存剩余为空&#34;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">);
            }
            DB</span>::<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">rollBack();
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> &#39;error&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
        } </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">catch</span> (<span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">Exception</span> <span data-mce-style="color: #800080;" style="font-family: &#34;Courier New&#34;; color: rgb(128, 0, 128);">$e</span><span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">) {
            </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);"> 出错回滚数据</span>
            DB::<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">rollBack();
            </span><span data-mce-style="color: #0000ff;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 255);">return</span> &#39;error&#39;<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">;
            </span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">//</span><span data-mce-style="color: #008000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 128, 0);">执行其他操作</span>
<span data-mce-style="color: #000000;" style="font-family: &#34;Courier New&#34;; color: rgb(0, 0, 0);">        }
    }

}</span></pre>


### AB测试

这里我使用了apache bench对代码进行测试

不了解的可以参阅这篇文章，有非常详细的讲解

https://www.jianshu.com/p/43d04d8baaf7

调用 代码中的 

AddUserToRedis()
方法将一堆请求用户放进redis队列中
先看库存![](https://images2018.cnblogs.com/blog/1191491/201807/1191491-20180705231035818-1775949487.png)

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;">这里设置了一千个库存<br/>开始压力测试</pre>

![](https://images2018.cnblogs.com/blog/1191491/201807/1191491-20180705231507169-588282292.png)

<pre style="color: rgb(0, 0, 0); font-family: &#34;Courier New&#34;; font-size: 12px; margin: 5px 8px; padding: 5px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-style: initial; text-decoration-color: initial;">向我们的程序发起1200个请求，并发量为200</pre>

![](https://images2018.cnblogs.com/blog/1191491/201807/1191491-20180705232539538-1369449251.png)

这里我们完成了1200个请求，其中标记失败的有1199个。这个是因为apache bench会以第一个请求响应的内容作为基准，

如果后续请求响应内容不一致会标记为失败，如果看到length中标记的数量不要方，基本可以忽略，我们的请求实际是完成了的。
