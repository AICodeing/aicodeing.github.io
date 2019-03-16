---
title: Android进程间通信-Binder之印象
date: 2019-03-16 13:26:34
tags:
	- Android
	- Binder

category: Android
comments: ture
---


说到`Android`中进程间通信，应用最广泛的非`Binder`莫属。  
在同一个进程空间中，内存虚地址的映射规则完全一致，他们在一个 虚地址空间中，所以两个函数互相调用很简单。但在两个不同的进程中，如我们的应用程序App和Framework中的`ActivityManagerService`进行函数调用，因为不在一个虚地址空间，所以没法直接通过内存地址访问到彼此的函数或变量。  
如下图:  

![](/img/binder/binder_mem_barrier.jpg "")  

既然无法`直接`访问到对方进程的内存空间，那只能通过`间接`方式了，binder的职责就是帮助进程间接访问对方进程空间。  

![](/img/binder/binder_desc.jpg "")

`Binder`是Android中使用最广泛的IPC机制，Binder 通信的组成元素有:  

```
1. Binder驱动
2. ServiceManager
3. Binder Client
4. Binder Server
```

Binder 通信可以类比我们常见的网络通信  
Binder 驱动相当于 网络请求中的路由器  
ServiceManager 相当于网络请求中的DNS解析服务器
Binder Client相当于网络请求的客户端
Binder Service 相当于 处理网络请求的服务器

本篇文章主要目的是给出Binder 通信的整体印象，后续文章中会详细分析Binder 通信中各个组成元素如何实现。

