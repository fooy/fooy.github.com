---
layout: post
title: "nodejs实现支持PAC的HTTP server"
date: 2016-04-01 00:00:00 +0800
comments: true
tag: [ http,node ]
---

HTTP代理服务支持[PAC](https://en.wikipedia.org/wiki/Proxy_auto-config)的好处是,可以根据URL动态的决定转发连接路径,节约不必要的路径开销

<!--more-->

( __update: 这个玩具项目源于当时翻墙的痛点：代理服务器无法根据域名执行动态的代理，即使国内的网站都要走墙外服务器绕一圈回来实在是很糟糕的连接体验
当时v2ray项目已经横空出世只是我没有发现，后来第一次打开v2ray的页面看到配置立刻理解了它的设计：v2ray中的"Freedom"出站协议不就是PAC中的"DIRECT"吗__)

写这段服务代码的初衷是辅助不支持pac的设备(比如我的小米手机)动态翻墙   
目前很多安卓设备都不支持PAC文件，在存在局域网代理服务的情况下，设备只能做到直连/代理二选一，
要么不停地开关代理，要么忍受通过代理服务访问国内网站的速度

很自然地想到能不能把根据PAC做路径选择的功能移到代理服务器端呢

搜索了一下，居然没有找到做到类似的功能的代理服务器，唯一接近的方案在[这里](https://serverfault.com/questions/198806/squid-selects-parent-depending-on-requested-url)

忍不住自己实现了一把，居然基本可用

PAC文件就是js文件，所以用node来实现最直观 ,从SwitchySharp中导出的pac文件直接可用

node提供的stream接口实现起来代码量如此之少,只需百行代码

[项目地址](https://github.com/fooy/pacproxy)

尝试把这个服务跑在我的刷了[asus-merlin](https://github.com/RMerl/asuswrt-merlin)的r7000上，因为[entware](https://github.com/Entware/entware)提供有nodejs 0.10.42的包可用

merlin自从回退了[这个修改](https://github.com/RMerl/asuswrt-merlin/commit/72792aea89cd562242e7d80317c45ac593f1bd0c)之后内存省出来好多，
nodejs不会再抛bad_alloc内存不够的异常了
毕竟路由器只有256M物理内存

具体使用场景如下:

1. 在路由器上启动`node pacproxy.js SwitchyPac.pac`
   * pacproxy.js 是pacproxy.coffee编译出的js，没必要在路由器上再装一个coffee包,内存实在吃紧
   * pacproxy.js 默认开启8080端口
   * SwitchyPac.pac从chrome插件选项中导出即可(大部分熟练的翻墙人都装有这个插件吧)
2. 设置手机代理服务器为http ://192.168.1.1:8000

赘述下工作原理就是pacproxy.js解析http proxy请求,执行SwitchyPac.pac中的FindProxyForURL函数：   
   - 返回DIRECT,就直接代理    
   - 返回SOCKS5://.. ,就进一步转发给路由器上的socks5服务(ss-local)端口   
   - 返回PROXY://..,就转发给其它http代理服务器   



其实这个项目可以做进一步扩展,考虑添加一个内置UI界面,可以提供类似Switchy的代理规则匹配和目标的动态修改.
