---
title: 在GoogleCloud上搭建SS代理服务
date: 2018-10-26 14:11:54
tags:
	- SS
	- Proxy
	- GoogleCloud
category: Tools
comments: true
---
GoogleCloud 目前有免费300$方案，该教程利用`GoogleCloud` 平台搭建 `Shadowsocks`服务。

#### 1. 注册GoogleCloud账户并登陆

登陆地址 [https://console.cloud.google.com](https://console.cloud.google.com)

账户建立过程忽略。

#### 2. 配置项目

进入 GoogleCloud 的后台。再开始搭建 `SS`服务之前需要先有一个`项目`如果已经有项目，直接选择，如果没有，按照下图新创建一个项目.

点击 顶部的 `Select a project`  
![](/img/ss_server/step_1.jpg "")   
![](/img/ss_server/step_2.jpg "")  
![](/img/ss_server/step_3.jpg "")  

项目创建完成之后，选择刚才创建的项目 然后 通过菜单栏选择 `Compute Engine`，然后进入到创建虚拟机实例 步骤。

#### 3. 创建VM实例  

![](/img/ss_server/step_4.jpg "")  
![](/img/ss_server/step_5.jpg "") 

创建虚拟机实例需要注意几个配置，具体看下图 红线部分

 ![](/img/ss_server/step_6_create_vm.jpg "") 
 
 __`vm`的名字:__ 可以随便起一个。
 
 __区域选择:__ 根据需求选择，一般原则是选择比较近的，例如 图示中选择的新加坡节点。
 
 __机器类型:__ 这个看需求，不过一般情况下 最低配置就能满足。
 
 __操作系统:__ 根据自己的情况，比如我习惯 Ubuntu 系统，就选择了 Ubuntu 18.0.4LTS
 
 剩下的默认配置就行。
 
 ![](/img/ss_server/step_6_vm_created.jpg "") 
 
 `VM`创建完成之后，然后先搁置起来，先去配置一些网络设置。
 
#### 4.网络设置

##### 创建防火墙规则

从左侧导航菜单选择 `VPC网络`->`防火墙规则`

![](/img/ss_server/step_7_create_fire_wall.jpg "")
 
![](/img/ss_server/step_7_begin_create_fire_rules.jpg "")
 
![](/img/ss_server/step_7_fire_rules_created.jpg "")

__防火墙名字:__ 这个随便起，见名之意即可

__流量方向:__ 选择入站

__目标:__ 选择 `网络中的所有实例`

__来源IP地址范围:__ `0.0.0.0/0`

__协议和端口:__ 全部允许

然后点击创建按钮完成防火墙配置。


##### 设置静态IP

从左侧导航菜单选择`VPC网络`-> `外部IP地址`

![](/img/ss_server/step_8_ip.jpg "")

![](/img/ss_server/step_8_begin_static_ip.jpg "")

![](/img/ss_server/step_8_static_ip_ok.jpg "")

静态IP 名称: 随便起一个，图例中是 `proxy-ip`

区域选择: 一定要选择你创建`VM`时 选择的区域和 `vm实例`


#### 5. 登陆 VM 实例，安装 SS 服务

进入 `VM` 实例 页面，通过 `SSH` 登陆 服务器

![](/img/ss_server/step_9_ssh_vm.jpg "")

![](/img/ss_server/step_9_link_vm.jpg "")

![](/img/ss_server/step_9_login_vm_ok.jpg "")

如图所示，成功登陆服务器。

由于 `SS 服务` 需要在 `root` 用户下登陆，下面我们修改一下 服务器的`root` 密码

![](/img/ss_server/step_10_reset_root_pwd.jpg "")

输入命令:

`sudo passwd root`  

然后设置 新的`root` 密码。
设置完之后 输入 命令:

`su`

然后输入密码进入 `root` 用户下

![](/img/ss_server/step_10_enter_root.jpg "")


以上步骤完成之后就开始我们的安装 操作了，执行如下命令:

```sh
wget --no-check-certificate -O shadowsocks-go.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-go.sh
chmod +x shadowsocks-go.sh
./shadowsocks-go.sh 2>&1 | tee shadowsocks-go.log
```
然后会让你配置一些信息，如图：

![](/img/ss_server/step_10_finish_ss_config.jpg "")

设置的密码即为你 `Shadowsocks`软件上需要填写的密码

端口号对应 你 `Shadowsocks`软件上需要填写的端口号

加密方式我们选择默认 `aes-256-cfb`即可。

一切配置完成之后，点击回车按钮，会自动安装服务，待服务安装完毕，会出现如下界面：

![](/img/ss_server/step_10_review_ss_config.jpg "")

这些就是你在 `SS`软件上需要填写的信息。










