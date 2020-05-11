---
title: "在R7000上配置SS + TPROXY + GEOIP透明代理"
date: 2016-06-18 00:00:00 +0800
comments: true
tags: [ network, proxy, router ]
---
花了一百块淘了一个很便宜的香港VPS , 几个月来连接服务还算稳定和速度, 于是想把透明代理搭起来，从此ipad上不会有一坨又一坨的404了


<!--more-->


目前我的ipad翻墙的解决方案是利用ios能支持加载pac文件:  
*   路由器上运行ss-local客户端连到vps,端口1080  
*   ipad网络连接-设置中指定一个从SwitchySharp导出的pac文件放在路由器http服务目录下如`http://192.168.1.1/wpad.pac`  
*   pac内容是基于gfwlist生成的js代码如`if (shExpMatch(url, "*://*reddit.com*")) return 'SOCKS 192.168.1.1:1080';`

然后这种基于白名单域名匹配还是有一定缺点  
*   经常遇到很多国外服务器页面打开很慢或者被墙但不在gfwlist里面,这时就要自己不停的去维护更新pac  
*   国产安卓手机是不支持pac的

其实大部分国外网站都很慢很卡，不如用iptables addon的`geoip`模块提供的过滤功能提供统一的快速通道 ,也就是用iptables定向本地所有的tcp/udp连接：   
   *   国内ip直接放行   
   *   国外ip定向到ss-redir服务端口进一步tunnel到vps服务器出    

配置过程如下:

*   首先我的路由器是刷了[XWRT-VORTEX](http://www.linksysinfo.org/index.php?threads/asuswrt-merlin-on-netgear-r7000.71108/)的R7000, 其实也就是 [asus-merlin](https://github.com/RMerl/asuswrt-merlin)

*   准备shadowsocks  
    因为会用到代理转发应用`ss-redir`,如果路由器上有entware-ng可以用如下命令安装:  
    `opkg install shadowsocks-libev`

*   准备geoip
    *   检查发现merlin现在的版本默认编译了geoip模块  {{< highlight bash >}}
admin@r7000:/tmp/home/root# find /lib/modules/|grep geoip
/lib/modules/2.6.36.4brcmarm/kernel/net/netfilter/xt_geoip.ko {{< /highlight >}}   
    *   下载IP索引数据库  
        下载[GeoIP数据库索引文件](/dl/geoip.tar) ,解压出geoipdb.idx, geoipdb.bin    
        把这两个文件拷贝到路由器/jffs/目录下   
        (__update: 如何重新编译这两个文件请参考文末的更新步骤__)
*   编译tproxy  
    tproxy模块主要用来转发udp(参考[这里](https://stackoverflow.com/questions/5615579/how-to-get-original-destination-port-of-redirected-udp-message))   
    发现merlin在发行版里没有开启tproxy,被迫开始折腾自己编译  ( __update__: 最新版merlin默认有最新模块此步骤非必要)  
    在merlin wiki上有配置firmware编译环境的[指导步骤](https://github.com/RMerl/asuswrt-merlin/wiki/Compile-Firmware-from-source-using-Ubuntu)  
    对于r7000, xwrt的维护者虽然劳苦功高十年如一日的把主干代码合进他的分支并提供firmware下载，但是处于一些考虑并没有开放分支代码.  
    所以我只好选择最接近的华硕型号rt-ac68u:

{{< highlight bash >}}
cd ~/asuswrt-merlin/release/src-rt-6.x.4708
make rt-ac68u
{{< /highlight >}}

所有的配置文件经过各种复杂的makefile规则处理一遍后,可以再次配置内核并编译tproxy模块,进入内核目录重新打开配置选项,  
把`CONFIG_NETFILTER_TPROXY`和`CONFIG_NETFILTER_XT_TARGET_TPROXY`设置为`M`
{{< highlight bash >}}
cd linux/linux-2.6.36
make menuconfig
{{< /highlight >}}

>  `<M> Transparent proxying support (EXPERIMENTAL) `
>  `<M> “TPROXY” target support (EXPERIMENTAL) (NEW)`

保存后重新编译内核
{{< highlight bash >}}
cd ~/asuswrt-merlin/release/src-rt-6.x.4708
make kernel
{{< /highlight >}}


安装模块

{{< highlight bash >}}
make -C ~/asuswrt-merlin/release/src-rt-6.x.4708/linux/linux-2.6.36 modules_install INSTALL_MOD_PATH=~/kmod/
{{< /highlight >}}

安装后发现tproxy模块已存在，将两个文件拷贝到路由器/jffs/目录

{{< highlight bash >}}
$ find ~/kmod |grep -i tproxy
/home/fooy/kmod/lib/modules/2.6.36.4brcmarm/kernel/net/netfilter/xt_TPROXY.ko
/home/fooy/kmod/lib/modules/2.6.36.4brcmarm/kernel/net/netfilter/nf_tproxy_core.ko
{{< /highlight >}}

*   添加启动脚本  
    geoip需要的索引文件默认路径在/var/geoip/目录，为重启后能正常工作可以修改启动脚本,在/jffs/scripts/init-start中加入:


{{< highlight bash >}}
mkdir /var/geoip
cp /jffs/geoipdb.idx /var/geoip
cp /jffs/geoipdb.bin /var/geoip

{{< /highlight >}}
>  这里是因为merlin的内核版本和模块都相对比较陈旧，geoip模块还是从/var/geoip/目录下读取ip索引文件

*   准备就绪之后就可以运行如下脚本把报文定向到ss-redir服务端口,然后非国内连接就会开启vps转接,注意本地ss-redir 和vps上的ss-server都要以`-u`参数启动，这样才能开启转发udp 调试阶段可以在两端使用`-v`参数查看详细代理IP打印输出 
*   这里再次声明`ss-redir`对于TCP代理使用的是REDIRECT模式 ,只有对UDP代理需要TPROXY模块支持

*   报文定向脚本

{{< highlight bash >}}
SS_SERVER_IP=XX.XXX.XXX.XXX # ss-server的ip地址
REDIR_PORT=1081
NO_REDIR="$SS_SERVER_IP 10.0.0.0/8 10.42.0.0/16 127.0.0.0/8 169.254.0.0/16 172.16.0.0/12 192.168.0.0/16"
echo "loading modules"
( lsmod | grep xt_geoip ) || insmod xt_geoip
( lsmod | grep TPROXY ) || ( insmod /jffs/nf_tproxy_core.ko && insmod /jffs/xt_TPROXY.ko )

echo "setup tcp redir"
iptables -t nat -N SS
iptables -t nat -F SS
for n in $NO_REDIR;do
iptables -t nat -A SS -d $n -j RETURN
done
iptables -t nat -A SS -p tcp -m geoip ! --destination-country CN -j REDIRECT --to-ports $REDIR_PORT
iptables -t nat -A PREROUTING -j SS

echo "setup udp redir"
ip rule add fwmark 0x01/0x01 table 100
ip route add local 0.0.0.0/0 dev lo table 100
iptables -t mangle -N SS
iptables -t mangle -F SS
for n in $NO_REDIR;do
iptables -t mangle -A SS -d $n -j RETURN
done
iptables -t mangle -A SS -p udp  -m geoip ! --destination-country CN -j TPROXY --on-port $REDIR_PORT --tproxy-mark 0x01/0x01
iptables -t mangle -A PREROUTING -j SS
echo "done"
{{< /highlight >}}

恢复iptables的脚本

{{< highlight bash >}}
del_in_table()
{
    iptables -t $1 -D PREROUTING -j SS
    iptables -t $1 -F SS
    iptables -t $1 -X SS
}
del_in_table nat
del_in_table mangle
{{< /highlight >}}

经过几天的使用感觉非常稳定,ss-redir占用内存很低  
透明代理最大的好处显而易见，房间里的电视盒子,pad,手机等等都自动翻墙了,看you2b什么的毫无压力了



( __update 1: 还要解决DNS污染问题__ )

以上步骤只是做了ip代理分流，并没有解决DNS污染的问题，如果在设备上直接把dns配置为非CN网段的DNS(8.8.8.8)当然是可以正常工作的，
但为了更好的体验还是要做DNS分流,步骤:

* 把某位好心人根据gfwlist转成dnsmasq的配置文件下载下来
{{< highlight bash >}}
curl https://cokebar.github.io/gfwlist2dnsmasq/dnsmasq_gfwlist.conf > dnsmasq_gfwlist.conf
{{< /highlight  >}}
* 把配置追加到dnsmasq的配置文件中并重启服务，在asus-merlin中是这么一个操作:   
(大部分主流路由器一般都是由dnsmasq接管本地的dns服务，我们可以在这一层做dns分流) 
{{< highlight bash >}}
# cat /jffs/scripts/dnsmasq.postconf 
#! /bin/sh
cat /jffs/dns/dnsmasq_gfwlist.conf >> $1

# service dnsmasq_restart
{{< /highlight  >}}
* 因为gfwlist名单中的域名都被分流到了本地的5353端口所以还要用ss-tunnel把这个端口代理到远程服务器上来转发:
{{< highlight bash >}}
/opt/bin/ss-tunnel -c ss_server.json -l 5353 -p 8080 -U -L 8.8.8.8:53
{{< /highlight  >}}

(所以路由器上最终需要ss-redir和ss-tunnel两个常驻进程配合使用)


( __update 2:如何更新IP索引数据库__ )   
merlin目前看起来是用的geoip模块的一个比较老的实现，估计以后早晚会升级到最新版本

*   merlin中重新编译geoipdb.idx和geoipdb.bin文件步骤：
    1.   下载编译csv2bin   
  {{< highlight bash >}}
wget http://people.netfilter.org/peejix/geoip/tools/csv2bin-20041103.tar.gz
tar xzvf csv2bin-20041103.tar.gz{{< /highlight >}}
  进入代码目录打开csv2bin.c文件找到CONTRYCOUNT宏改为一个比较大的数字这样程序才不会数组越界崩溃，然后编译  
  `< #define COUNTRYCOUNT 241`   
  `> #define COUNTRYCOUNT 600`
  {{< highlight bash >}}make csv2bin{{< /highlight >}}
    2.   下载IP数据库  
    到`https://sourceforge.net/projects/xtables-addons/files/Xtables-addons/`下载最新版 ,比如xtables-addons-2.14.tar.xz
    解压后运行:  
    {{< highlight bash >}}xtables-addons-2.14/geoip/xt_geoip_dl{{< /highlight >}}
    就会得到`GeoIPCountryWhois.csv`
    3.   生成数据库索引
    {{< highlight bash >}}./csv2bin GeoIPCountryWhois.csv{{< /highlight >}}
*   我也试着在openwrt中尝试了以上解决方案，基本原理是一样的，geoip模块和xtproxy模块都可以直接安装不用自己编译，这里也列出适用于openwrt的geoip模块数据库生成办法:
    1.   安装geoip和tproxy模块  
  使用opkg查找安装即可:
  {{< highlight bash >}}
$ opkg list-installed |grep geoip
iptables-mod-geoip - 2.5-1
kmod-ipt-geoip - 3.18.23+2.5-1
$ opkg list-installed|grep tproxy
iptables-mod-tproxy - 1.4.21-1
kmod-ipt-tproxy - 3.18.23-1{{< /highlight >}}
    2.   下载IP数据库(同以上步骤2)  
    3.   生成IP数据库索引  
    生成工具是perl语言，所以需要安装一些依赖，在mac下的命令如下，linux系统没测过
    {{< highlight bash >}}
#install perl module for xt_geoip_build
sudo cpan App::cpanminus
sudo cpanm Text::CSV_XS
./xt_geoip_build GeoIPCountryWhois.csv
{{< /highlight >}}
    4.   安装到路由器   
    {{< highlight bash >}}
ssh root@openwrt mkdir -p /usr/share/xt_geoip/
scp -r BE root@openwrt:/usr/share/xt_geoip/
{{< /highlight >}}


( __update 3: 使用ipset替换geoip__ )   

如果觉得geoip模块麻烦可以使用ipset来替代,加载CN网段的参考命令如下:
    {{< highlight bash >}}

#get cn zone ipset
wget http://www.ipdeny.com/ipblocks/data/countries/cn.zone

#load ipset
ipset create cnblock nethash
cat cn.zone |while read c;do ipset add cnblock $c; done

#简单测试一个IP
ipset list cnblock
ipset test  cnblock 8.8.8.8
{{< /highlight >}}
然后在脚本中替换geoip模块的匹配为下面的语句:
    {{< highlight bash >}}
iptables -t mangle -A SS -p udp  -m set ! --match-set cnblock dst -j TPROXY --on-port $REDIR_PORT --tproxy-mark 0x01/0x01
{{< /highlight >}}

