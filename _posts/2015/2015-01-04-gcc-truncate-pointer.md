---
layout:post
title:gcc截断指针长度
categories:
- 技术
tag:
- gcc
---
1.问题
今天在学习cpython的时候，发生了一个很诡异的问题。问题简单描述如下：
在文件a.c中调用了b.c中的一个方法
> PyStrObject * object = PyStr_Create(str);
发现当PyStr_Create()返回地址过长时，a.c中得到的结果会把超过的4个字节给截断。
举个例子，在b.c中,PyStr_Create()方法返回了0x7fc963401a80，但是在a.c中得到的结果却是0x63401a80。
2.测试
看到这个结果后，我于是在旁边的测试文件中测试了一下，注意，我这里把函数定义和调用写在了同一个文件中，结果调用和返回结果一切正常，没有发生截断现象。

难道是gcc链接时的问题？我又把函数定义和函数使用放在了2个不同的文件中，果然发生了上文中的截断现象。

3.
去网上搜了一下，在这里找到了答案[linux GCC编译和使用方法](http://cjhust.blog.163.com/blog/static/175827157201492934526569/)。
3.1 解决方案
重新声明一个.h文件包含PyStr_Create()的声明，然后a.c引用这个.h
3.2 警告
关于上个链接中提到的警告，我没有找到，我这里的警告是这样的
> incompatible integer to pointer conversion initializing 'char *' with an expression of type 'int' [-Wint-conversion]
也就是说gcc编译器把没有声明的PyStr_Create的返回值当成了int。
