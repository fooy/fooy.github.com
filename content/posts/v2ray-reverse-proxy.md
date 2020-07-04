---
title: "使用v2ray反向代理配置家庭云桌面"
date: 2020-06-30 00:00:03 +0800
comments: true
tags: [ v2ray ]
---

<!--more-->

翻看v2ray的配置教程发现了反向代理功能, 虽然对v2ray的强大功能还在理解深化中，当看到 [反向代理2](https://guide.v2fly.org/app/reverse2.html)，我觉得这个功能真是酷炫，配置小火箭完全可以实现很顺滑的"家庭云体验"，立刻动手把官网的配置拷贝实验了一把.

这种穿透+反向动态代理的功能用其它的软件比如`frp`也能实现类似的效果,这里我觉得v2ray也不弱，和小火箭强大的协议连接能力配合度很高，真正可以做到"一键回城"


1. 首先服务端和穿透发起端的配置直接拷贝官方配置就可以了,VPS服务器尽量保证带宽和时延.

2. 在小火箭端创建一个新的连接，这里命名这个连接为HOME , 也就是负责访问家庭网络的连接（配置文件中的"C"）

![](/images/reverse1.png)

3. 在小火箭的正在使用的配置文件中检查去除对家庭网段的旁路配置，也就是配置家庭网段的数据包走隧道中转，这里我的局域网段是192.168.1.x ,
点击配置文件-编辑配置-通用部分，把"跳过代理(http)"和"旁路TUN(tcp/ip)"中覆盖到192.168.1.x的网段去掉,包括比如192.168.0.0/16
![](/images/reverse3.png)

4. 同时编辑配置文件的规则部分，添加规则, 加入匹配家庭网段的IP-CIDR，并指示转发连接为新建的连接"HOME" ,这里PROXY 其实就是当前小火箭的默认连接，
而HOME是家庭网段的连接, 可以看到小火箭是支持多个连接的嵌套转发的，可以定义非常丰富的规则，本篇的重点只是是把代理家庭网段的规则加入其中.
![](/images/reverse2.png)


5. 在加入规则并且连接生效的情况下，iOS中常用的应用在移动连接环境下已经可以访问家庭的网络设备，举例说明:
 - nplayer已经可以浏览到家庭samba服务器的文件并且能够播放
 - chrome可以访问家庭网盘的http文件服务器上的pdf
 - vnc viewer可以连接到家庭的笔记本登录后可以打开IDE敲代码。。  

  同时从上图三条规则可以直观看到小火箭灵活的配置能力，访问家庭的网络不影响正常的互联网访问.


6. 一点不足是目前还没有找到有效的办法实现家庭设备的域名解析，好在局域网设备的IP地址一般不变，在小火箭规则中加入hosts映射即可.

========================== 亲测有效的配置文件   
服务端:
{{< highlight json >}}
{
  "reverse":{  //这是 B 的反向代理设置，必须有下面的 portals 对象
    "portals":[
      {
        "tag":"portal",
        "domain":"private.cloud.com"        // 必须和上面 A 设定的域名一样
      }
    ]
  },
  "inbounds":[
    {
      // 接受 C 的inbound
      "tag":"tunnel", // 标签，路由中用到
      "port":443,
      "protocol":"vmess",
      "settings":{
        "clients":[
          {
            "id":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "security": "auto",
            "alterId":64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
	    "security": "none",
        "wsSettings": {
	      "path": "/vms"
        }
      }
    },
    // 另一个 inbound，接受 A 主动发起的请求
    {
      "tag": "interconn",// 标签，路由中用到
      "port":8993,
      "protocol":"vmess",
      "settings":{
        "clients":[
          {
            "id":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "security": "auto",
            "alterId":64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
	    "security": "none",
        "wsSettings": {
	      "path": "/vms"
        }
      }
    }
  ],
  "routing":{
    "rules":[
      {  //路由规则，接收 C 的请求后发给 A
        "type":"field",
        "inboundTag":[
          "interconn"
        ],
        "outboundTag":"portal"
      },
      {  //路由规则，让 B 能够识别这是 A 主动发起的反向代理连接
        "type":"field",
        "inboundTag":[
          "tunnel"
        ],
        "domain":[
          "full:private.cloud.com" // 将指定域名的请求发给 A，如果希望将全部流量发给 A，这里可以不设置域名规则。
        ],
        "outboundTag":"portal"
      }
    ]
  },
  "log":{
    "access": "/tmp/v2ray.access.log",
    "error": "/tmp/v2ray.error.log",
    "loglevel": "info"
  }
}
{{< /highlight >}}

家庭端:
{{< highlight json >}}
{
  "reverse":{
    // 这是 A 的反向代理设置，必须有下面的 bridges 对象
    "bridges":[
      {
        "tag":"bridge", // 关于 A 的反向代理标签，在路由中会用到
        "domain":"private.cloud.com" // A 和 B 反向代理通信的域名，可以自己取一个，可以不是自己购买的域名，但必须跟下面 B 中的 reverse 配置的域名一致
      }
    ]
  },
  "outbounds": [
    {
      //A连接B的outbound
      "tag":"tunnel", // A 连接 B 的 outbound 的标签，在路由中会用到
      "protocol":"vmess",
      "settings":{
        "vnext":[
          {
            "address":"my-server.com", // B 地址，IP 或 实际的域名
            "port":8993,
            "users":[
              {
                "id":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                "security": "auto",
                "alterId":64
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/vms"
        }
      }
    },
    // 另一个 outbound，最终连接私有网盘
    {
      "protocol":"freedom",
      "settings":{
      },
      "tag":"out"
    }
  ],
  "routing":{
    "rules":[
      {
        // 配置 A 主动连接 B 的路由规则
        "type":"field",
        "inboundTag":[
          "bridge"
        ],
        "domain":[
          "full:private.cloud.com"
        ],
        "outboundTag":"tunnel"
      },
      {
        // 反向连接访问私有网盘的规则
        "type":"field",
        "inboundTag":[
          "bridge"
        ],
        "outboundTag":"out"
      }
    ]
  }
  ,
  "log":{
    "access": "/tmp/v2ray.access.log",
    "error": "/tmp/v2ray.error.log",
    "loglevel": "debug"
  }
}
{{< /highlight >}}

