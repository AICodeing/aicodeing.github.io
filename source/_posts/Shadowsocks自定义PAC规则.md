---
title: Shadowsocks自定义PAC规则
date: 2018-10-17 11:14:28
tags:
	- SS
	- PAC
category: Tools
comments: true
---
科学上网，SS是一大利器。它的实现原理 就是墙外一台服务器，墙内一台服务器(也就是我们的客户端)，两台服务器之间加密通讯，防火墙因为无法判定用户访问的网站是否在防火墙列表，所以放行,这样就可以通过墙外服务器做代理啦。但我们知道，任何网站都通过墙外服务器是`不明智的`，例如我们访问`baidu.com`，百度在国内架设了很多服务器，一般都是就近访问的，如果也通过代理到国外绕一圈，得不偿失。所以就需要一个忽略列表，或者一个需要走代理的列表，而PAC 规则就是来实现这个的。

### 语法  
PAC的配置规则语法和GFWlist 相同，即[adblock](https://adblockplus.org/en/filter-cheatsheet)   

下面我引用 `adblock`的示例来做说明。

#### 1. 匹配地址中的一部分内容(通配符匹配)  
比如我们想匹配以下地址:

- http://example.com/banner/foo/img
+ http://example.com/banner/foo/bar/img?param
* http://example.com/banner//img/foo

但是不想匹配这些地址:

- http://example.com/banner/img
- http://example.com/banner/foo/imgraph
- http://example.com/banner/foo/img.gif

那么我们可以写这样的匹配规则:  

__`/banner/*/img^`__  

通配符`*` 表明中间可以是任意路径或字符，但是必须要有,如果没有，就必须出现两个`/` 如`http://example.com/banner//img/foo`  

符合 `^`  表明 地址必须在这里结束，或者后面跟着的是 `/`，或是参数字符，例如`?`。  

#### 2. 匹配绝对地址  
 
比如我就像匹配地址:  

- http://www.baidu.com

那么匹配规则可以这样:  

__`|http://www.baidu.com|`__  

在该规则下 以下地址就无法匹配:  

- http://www.baidu.com/logo.jpg

#### 3. 匹配开头或结尾  

比如我想匹配以 `http://example.com` 开头的网址,例如:  

- http://example.com/index.html
- http://example.com/us/index.html

那么匹配规则为:   

__`|http://example.com`__  

如果匹配以 `example.com`结尾的网址,例如:  

- http://sandbox.example.com
- http://www.example.com
- http://www.beta.example.com

那么匹配规则为:  
__`example.com|`__  

#### 4. 通过域名匹配  
比如我想匹配所有来自 `ads.example.com` 这个域名的网址  

- http://ads.example.com/foo.gif
- http://server1.ads.example.com/foo.gif
- https://ads.example.com:8000/

而不想匹配:  

- http://ads.example.com.ua/foo.gif
- http://example.com/redirect/http://ads.example.com/

那么匹配规则为: __`||ads.example.com^`__ 

#### 5. 高级匹配规则  

假如我有这样的需求:

1. 我只匹配来自 example.com 这个域名或其子域名的地址;
2. 但不匹配 foo.example.com 这个子域名已经它的子域名
3. 只匹配 图片或js文件的请求地址

那么这个匹配规则为:

__`||example.com^$script,image,domain=example.com|~foo.example.com`__  

那么 地址 `http://ads.example.com/foo.gif` 就能被匹配上。

#### 6. 排除在规则之外  

比如我不想让 `http://ads.example.com/notbanner/1.png`通过匹配
但是希望 `http://ads.example.com/notbanner/`下的所有 js 文件通过匹配

那么匹配规则:  

__`@@||ads.example.com/notbanner^$~script`__

##### 排除整个网站的网址

比如要排除 所有 域名`example.com`的网站

那么匹配规则如下:  

__`@@||example.com^$document`__

假如我们想注释掉一个规则是否支持呢？    
答案是__肯定的__。  

例如: `!|http://example.com`  

### SS 中编辑
下面 我以 `ShadowsocksX-NG`-1.8.2 版本为例来说明一下如何编辑自定义`PAC规则`

![](/img/ss/ss_1.jpg "")

![](/img/ss/ss_2.jpg "")

配置完之后最好点击一下 `从GFW List 更新PAC` 按钮。









