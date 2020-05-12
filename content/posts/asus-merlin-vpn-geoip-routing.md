---
title: "在asus-merlin中配置vpn + policy routing实现策略上网"
date: 2018-06-30 00:00:00 +0800
comments: true
tags: [ vpn, router ]
---

摘要:使用geoip来标记国/内外的IP地址，并用策略路由来分流到VPN.

<!--more-->

[上一篇](../setup-transparent-proxy-on-r7000/)介绍过使用iptables配合透明代理+SS来动态转发tcp/udp的解决方案，但其中涉及到到工具SS和透明代理的概念对于很多人并不是那么熟悉和直观

其实上一篇的解决方案完全可以用VPN+策略路由来代替  
策略路由的好处是工作在IP层, 配合GEOIP对不同目的地址的包进行筛选标记并路由, 可以做到自定义的策略上网

基本原理:
从iptables的[工作图](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)
可以看出, IP包在经过路由决策之前会经过mangle表的PREROUTING链,我们只需要在PREROUTING链中根据一定的条件标记墙内外的IP地址,
然后再配置路由策略根据不同的mark做出相应的路由决策即可


下面是具体步骤:

1. 创建VPN连接  
    这里我建立的是pptp类型的vpn连接，其它连接类型(openvpn,l2tp/ipsec)没有进一步实验，基本原理是一样的

    asus-merlin的web UI界面支持建立pptp的VPN连接,当连接建立成功后通过命令`ip addr`可以看到新建立的ppp5接口

    ![](/images/vpnc.png)

2. 使用geoip模块来标记非中国地区的IP  
    这里和上一篇提到的一样使用geoip模块，需要先下载数据文件[geoipdb.bin和geoipdb.idx](dl/geoip.tar), 拷贝到路由器上/var/geoip/目录

    然后使用如下命令标记non-china IP的mark为'2':
    {{< highlight bash >}}
iptables -t mangle -A PREROUTING -i br0 -m geoip ! --destination-country CN  -j MARK --set-mark 2
{{< /highlight >}}

3. 配置policy routing,转发被标记的IP包到VPN连接  
    使用如下命令把mark为'2'的包交给table 80(任选一个空路由表)处理
然后配置table 80的默认路由为从vpn接口转发  
    {{< highlight bash >}}
ip rule add fwmark 2 table 80
ip route add default table 80 dev $vpn_if
{{< /highlight >}}

4. 为dnsmasq配置DNS路由  
    策略路由配置完后要考虑DNS的问题  
    (__update: 这里是当时为了简便直接配置DNS服务器的请求走VPN路由出去绕过DNS污染，
    但是这种方式很不推荐无法解决墙外的DNS服务器返回大量不可用CDN的问题，所以这里还是要使用gfwlist+dnsmasq+ipset来搭建自己的DNS分流配置,
    详情可以搜索相关文章...__)   
    通常情况下局域网内的设备的DNS请求是交给dnsmasq代理的(当然也可以配置dhcp下发自定义的DNS)    
    路由器会在每次连接ISP成功后把ISP提供的DNS服务器列表设置为dnsmasq的 __上游DNS服务器__   
    并且,当连接vpn成功后,路由器也会把vpn服务器分配的DNS更新为dnsmasq的上游服务器  
    不管怎样，需要考虑的是，用户的DNS请求(具体是dnsmasq发起的UDP连接)走的是哪个路由转发路径  
    而且dnsmasq向上游服务器的请求是从路由器用户空间发出的，不会经过mangle表, 所以需要为DNS配置额外的路由  
    这里，为了防止DNS污染, 我在默认路由表中配置DNS走vpn线路  
    {{< highlight bash >}}
for s in `nvram get vpnc_dns`;do
    ip route add to $s dev $vpn_if
done
{{< /highlight >}}

5. 验证  
    当一切配置完毕就可以在局域网设备上验证路由是否正确了,  
    我在mac上用traceroute命令验证  
    可以看到qq.com和baidu.com等中文网站走的是ISP线路(10.\*.\*.\*)
    ![](/images/r1.png)   
    youtube.com走的是vpn线路(192.168.2.\*)
    ![](/images/r2.png)   

    验证iptables命令: `iptables -t mangle -L -v`  
    验证routing命令: `ip route show table 80` , `ip rule list`

6. 将配置命令加入ip-up-script中  
    asus-merlin的vpn客户端实现都在这个[vpnc.c](https://github.com/RMerl/asuswrt-merlin/blob/master/release/src/router/rc/vpnc.c)文件里，  
可以看到每当vpn连上之后会重新覆盖默认路由,  
因为vpn客户端会有多次重新连接的场景我们需要把路由配置脚本注入到vpn连接成功后的执行脚本中/tmp/ppp/vpnc-ip-up(查看/tmp/ppp/vpnc_options.pptp)  
也就是每次ppp连接成功后执行策略路由脚本

7. 完整的脚本  
    使用: 在web UI中连接vpn之前先运行 `. /jffs/vpn.sh hook`  
    {{< highlight bash >}}
vpn_if=$(nvram get vpnc_pppoe_ifname)
vpn_gw=$(nvram get vpnc_gateway)

PASS_MAC_LIST=""

vpn_clear()
{
ip rule list | grep 'lookup 80' | while read line;do ip rule del lookup 80;done
ip route flush table 80
iptables -t mangle -D PREROUTING -j VIA_VPN
iptables -t mangle -F VIA_VPN
iptables -t mangle -X VIA_VPN

for s in `nvram get vpnc_dns`;do
    ip route del to $s dev $vpn_if
done
ip route flush cache
}

vpn_set()
{
if [ ! `nvram get vpnc_state_t` == "2" ] ;then
  echo "VPN not connected";
  return;
fi

( lsmod | grep xt_geoip ) || insmod xt_geoip

echo 0 > /proc/sys/net/ipv4/conf/$vpn_if/rp_filter

#added by vpnc.c when connected,need to delete first
ip route del default dev $vpn_if

#dnsmasq upstream (updated in /tmp/resolv.dnsmasq when connected) request shall go through VPN
for s in `nvram get vpnc_dns`;do
    ip route add to $s dev $vpn_if
done

#add default routing in table 80
ip route add default table 80 dev $vpn_if

#add ip rule
ip rule list | grep 'lookup 80' | while read line;do ip rule del lookup 80;done
ip rule add fwmark 2 table 80

#add iptables rule
iptables -t mangle -D PREROUTING -j VIA_VPN
iptables -t mangle -N VIA_VPN
iptables -t mangle -F VIA_VPN
for m in $PASS_MAC_LIST;do
    iptables -t mangle -A VIA_VPN -i br0 -m mac --mac-source $m -j RETURN
done
iptables -t mangle -A VIA_VPN -i br0 -m geoip ! --destination-country CN  -j MARK --set-mark 2
iptables -t mangle -A PREROUTING -j VIA_VPN
ip route flush cache
}

vpn_hook()
{
mkdir -p /tmp/ppp/ppp
ln -sf /sbin/rc /tmp/ppp/ppp/vpnc-ip-up
rm /tmp/ppp/vpnc-ip-up
cat > /tmp/ppp/vpnc-ip-up <<!
#! /bin/sh
/tmp/ppp/ppp/vpnc-ip-up
logger -t vpnc "vpn client connected"
. /jffs/vpn.sh set
!
chmod +x /tmp/ppp/vpnc-ip-up
}

eval "vpn_"$1
    {{< /highlight >}}

8. TODO
    - 可以用ipset来替代geoip
    - 当有多个vpn连接时可以在路由表中加入相应的多条默认路由实现负载均衡


参考:    
    - http://linux-ip.net/html/adv-multi-internet.html  
    - https://superuser.com/questions/1185861/linux-routing-based-on-domain-names/
