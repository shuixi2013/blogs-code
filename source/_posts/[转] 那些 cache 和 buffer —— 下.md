title: (转) 那些 cache 和 buffer —— 下
date: 2015-01-31 17:40:16
updated: 2015-01-31 17:40:16
categories: [Basics Knowledge]
tags: [basics]
---

[前面一篇文章](http://light3moon.com/2015/01/31/[转] 那些 cache 和 buffer —— 上 "前面一篇文章") 谈了一些Linux系统层面的cache和buffer。这里主要谈谈应用层面的那些cache。相比系统层面的cache集中在IO上，应用层面的cache就显得五花八门了。就从WEB说起吧。

## web服务器缓存

web缓存对于服务器和客户端都是不可或缺的。对于web服务器来说，缓存是非常重要的东西，它可以大大的增加并发，缩短相应时间，减少服务器负荷。原理很简单。因为对于一个URL来说，很短时间内它的内容变化其实是不大的，如果每次请求都要服务器算一遍就显得太浪费了。所以web缓存就是放在web服务器前面的一个代理，它接收用户请求，并向后端请求，在返回响应的时候将这些响应缓存起来，遇到请求时不经过服务器计算直接返回响应，从大大提高响应速度，尤其是在请求量很大的时候。web缓存代表作是squid和varnish。

## 客户端缓存

对于客户端的浏览器来说缓存同样不可或缺。甚至在HTTP协议中都为缓存提供了支持，HTTP返回码304 Not Modified就是告诉浏览器，你要的内容没变，用你缓存中的吧。在HTTP协议之外，浏览器自己也会做许多的缓存，对于图片啊，js什么的，短时间内的请求就自作主张直接不去远程取了，会大大减少请求量，从而节省大量的时间，用户需要速度。况且浏览器和服务器本是同根生嘛，不必相互煎熬。所以有时候你得强制刷新。
与web服务器紧密相关的就是数据库了。

## 数据库缓存

在数据库系统中，缓存同样无处不在。因为同样的道理，对同一个sql查询来说，在某些条件下（比如它查的表自上次查询后都没更新过）它的结果就应该和上次查询一样的。于是mysql提供了query cache。也有些框架提供了缓存功能，比如hibernate。这都是读缓存，目的在于读很多，而写比较少的时候提高读的性能，如果写很多，而读比较少的话这类缓存就没什么用了。于是，在一些情况下我们希望可以为数据库引入写缓存。典型的是主键查询和更新。于是出现了kv数据库（比如memcached）。可以提供基于主键查询的读写缓存。这对于提高数据系统的整体性能是极其重要的。

## 其他缓存

其他五花八门的cache还有那些呢？

比如dns缓存，dns缓存同样存在于客户端和dns服务器中，与web缓存的原理是一样的。将dns的解析结果缓存起来。
比如arp缓存，将arp的结果缓存起来。

甚至，连编译系统也引入了缓存比如ccache，比如visual studio中的pch（预编译头）机制。

最后，我想说的是：cache is king! cache is everywhere!

[原始出处](http://blog.dccmx.com/2011/06/about-cache-and-buffer-2/ "原始出处")


