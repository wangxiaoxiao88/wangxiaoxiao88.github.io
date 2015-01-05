---
layout: post
title: httpclient乱码
category: 技术
---
昨天更新遇到了一个很诡异的问题，一个服务接口在测试机上运行ok，但是当更新到线上去后，出现乱码。  

先说下这个接口的内容，主要是用httpclient发送了一个post请求，然后对结果进行解析。乱码就是发生在结果进行解析的时候。奇怪的一点是post请求中返回的结果中没有涉及到中文，**是纯英文的**。我的排查过程如下:

###问题分析

1.排除tomcat问题

我们的这个服务是运行在tomcat服务器中的，因此我绕过tomcat服务器，在问题机器上直接写了一段java程序，运行后发现还是出现乱码，同样的程序在测试机上运行ok。  

2.socket问题  

此时把socket接受的数据打印出来发现，问题机器和运行ok的机器打印出来的byte数组不一样，连长度都不一样。怀疑系统配置问题导致socket接受异常，但是实在没有理由相信这个。。。  

3.httpclient问题  

此时只能去怀疑httpclient，把Httpclient的log通过trace级别打出，发现问题机器比运行ok的机器打出的log多了3行。开始找为什么会多出这3行？发现多了以下内容:  

> DEBUG 2013-06-18 10:45:52.46 [httpclient.wire.header] >> "Accept-Encoding:  gzip, deflate[\r][\n]"
 
真相大白了，原来是Accept-Encoding在搞鬼...它的作用可以看[HTTP协议之Content-Encoding](http://guojuanjun.blog.51cto.com/277646/667067).  

4.进一步分析  

为什么测试环境和线上环境不一致呢？线上环境的nginx中都配置了

> gzip  on;  
> gzip_http_version 1.0;

而测试环境中，不知道被谁修改了hosts文件，使得post请求的域名指向了另外一台机器，而恰恰这台机器上的nginx没有配置以上属性，从而导致没有gzip压缩。  

###解决方案

判断返回的httpheader中是否包含Accept-Encoding,包含且值是gzip的话，进行解压缩，解压缩的代码如下：
{% highlight java %}
 GZIPInputStream gzip = new GZIPInputStream(post.getResponseBodyAsStream());
 ByteArrayOutputStream baos = new ByteArrayOutputStream();
 int len = 0;
 while ((len = gzip.read(body)) != -1) {
        baos.write(body, 0, len);
 }
{% endhighlight %}
