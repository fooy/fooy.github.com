---
layout: post
title: "UDP透明代理的报文重定向"
date: 2016-06-17 00:00:00 +0800
comments: true
tag: [ linux,udp,tproxy ]
---

这两天想把路由器上的透明代理搭起来，在搜索了一些使用iptables配合ss-redir的命令之后，我被一个疑问困住了：   
**iptable在nat表PREROUTING链对协议报文做重定向之后，代理服务是如何知道报文的目的地址/端口的?**   

<!--more-->

显然代理服务收到的报文的ip/port是被修改后的（一般是local ip/port）,如果不知道目的地址如何forward呢?        

事实linux是可以做到的，内核网络netfilter框架 [connection tracking](https://en.wikipedia.org/wiki/Netfilter#Connection_tracking)功能负责 维护着进出连接的5员组信息,当代理应用收到一个包时，只要通过修改后的5员组查找内核的conntrack， 就能够找到对应原始连接5员组中dest ip/port(可以参考命令cat /proc/net/ip_conntrack)     

netfilter专门实现了接口getsockopt(`SO_ORIGINAL_DST`)提供这一查找功能.         

然而目前内核中对SO_ORIGINAL_DST支持只限于tcp和scp,为什么不支持udp ? 参考这个 [解释](http://lists.netfilter.org/pipermail/netfilter-devel/2001-February/000564.html) 和[SO问答](https://stackoverflow.com/questions/5281409/get-destination-address-of-a-received-udp-packet)，   

总结主要问题在于:
- 对于tcp可以在每个连接建立时通过accept()获得一个新的文件描述符(用来引用这个连接),然后后续调用getsockopt(SO_ORIGINAL_DST)在conntracking中查找IP原始目的地址， 但是对于无连接的udp,协议栈无法提供这种语义操作.
- connection tracking 确实会对不同的UDP连接也会建立一个tracking ,但是这个conntrack很容易超时,对于同时两个来自同一个源但不同目的地址的nat连接在conntrack表中就无法查找处理了，参见[这里](http://lists.netfilter.org/pipermail/netfilter-devel/2001-July/001524.html)



所以目前推荐的做法是对udp使用TPROXY配合转发,要点如下:   
1. 在mangle表里通过TPROXY对报文加上fwmark，TPROXY并不会修改报文头  
2. 使用`ip rule`修改路由决策，把打上fwmark的报文重定向到透明代理服务socket.  
3. 代理socket需要setsockopt(`IP_TRANSPARENT`)才能收到处理发送非本机IP/Port的报文  
4. 代理socket需要setsockopt(`IP_RECVORIGDSTADDR`),这样recvmsg收到的udp辅助报文头中包含原始dest ip/port信息


以上理论是基于资料和grep下[shadowsocks](https://github.com/shadowsocks/shadowsocks-libev)代码

有理论也要有实做下一篇要晒下在我的r7000上配置透明代理的过程

其它参考  
* [kernel patch to add UDP](http://lists.netfilter.org/pipermail/netfilter-devel/2001-July/001522.html)  
* [china unix](http://bbs.chinaunix.net/thread-4191882-1-1.html)  
* [tproxy kernel doc](https://www.kernel.org/doc/Documentation/networking/tproxy.txt)  
* [how-to-get-original-destination-port-of-redirected-udp-message](https://stackoverflow.com/questions/5615579/how-to-get-original-destination-port-of-redirected-udp-message)    
* tproxy help: `iptables -j TPROXY -h`  
* `man 7 ip`
