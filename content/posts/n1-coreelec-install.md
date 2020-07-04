---
title: "CoreElec安装记录(n1折腾笔记)"
date: 2020-06-30 00:00:01 +0800
comments: true
tags: [ CoreElec ]
---

<!--more-->
疫情期间不用出门所以又在拼夕夕上买入两个n1盒子来回折腾，一个刷安卓系统用来连接附属屏幕可以用来随时看you2be+投屏来节省笔记本cpu，另一个刷CoreElec做为24小时开机的网盘+接电视高清解码.

CoreElec完全取代了之前树莓派3B的位置，能播放大量树莓派没有能力播放的文件什么1080p,x265等全部秒杀.
盒子到来之前又迫不及待的把冲动消费后闲置许久的appleTV 4K在闲鱼出手同时换手一块新的显示屏作为连n1的副屏幕,考虑n1盒子百元的价格来说，真可谓为了这盘醋才包了这锅饺子...


#### 编译安装

这里使用RuralHunter提供的代码库，此库加入了一些bugfix和编译所需的命令:
{{< highlight bash >}}
git clone https://github.com/RuralHunter/CoreELEC.git
cd CoreELEC/
{{< /highlight >}}

由于我的一个磁盘是经过luks加密的需要内核支持，自己编译的好处就在于可以定制需要的功能，
搜索发现已经有人向官方库提交过对应的[PR](https://github.com/CoreELEC/CoreELEC/pull/230/commits)
把对应的修改拉下来:

{{< highlight bash >}}
git checkout origin/n1-9.2 -b build
git remote add coreelec https://github.com/CoreELEC/CoreELEC.git
git fetch coreelec pull/230/head:support_luks
git cherry-pick support_luks
#用户空间工具可以用entware安装opkg install cryptsetup
{{< /highlight >}}

编译生成可烧录在u盘的img文件,这个过程要持续很久。。
{{< highlight bash >}}
./mkn1
#output to ./target/CoreELEC-Phicomm-N1.arm-9.2.1.img.gz
{{< /highlight >}}

RuralHunter提供的代码库目前还不支持安装到EMMC和对应的EMMC/U盘双启动脚本支持,所以我整理了从恩山拿到的另一个版本的发行版中对应需要的文件: 
把[启动脚本和安装到emmc脚本](/dl/coreelec_boot.zip)覆盖到编译生成的img镜像根目录下.

u盘启动后运行`/flash/installtoemmc`安装系统到内置存储
#### 加入hd-idle

其实这个n1盒子平时主要的角色是7x24简易网盘，我习惯性使用hd-idle让常年挂在usb口的移动硬盘尽量保持安静.
在entware中查找不到hd-idle，在CoreElec代码目录下搜索发现hd-idle被包含在system-tools包.
这里又需要自己编译:
{{< highlight bash >}}
PROJECT=Amlogic ARCH=arm DEVICE=Phicomm-N1 CUSTOM_VERSION=9.2.1 ./scripts/create_addon system-tools
{{< /highlight >}}

编译完成后可以把生成的插件包拷贝到系统中并在kodi界面上使用浏览插件选项安装(其实可以直接解压到某处)
然后在`.config`目录下创建一个systemd unit文件以便开机启动:
{{< highlight bash >}}
CoreELEC:~/.config/system.d # cat hd-idle.service
[Unit]
Description=hd-idle - spin down idle hard disks
Documentation=man:hd-idle(8)

[Service]
Type=forking
ExecStart=/storage/.kodi/addons/virtual.system-tools/bin/hd-idle -i 0 -a sda -i 600 -a sdb -i 600 -a sdc -i 600

[Install]
WantedBy=multi-user.target
{{< /highlight >}}


#### 开启raid

为了进一步让这个网盘盒子能够有一定的数据保护,我又编译开启了内核的RAID支持以及管理工具mdadm, [这里](../coreelec-raid10/)单独记录一下.

#### 好用的工具及功能
- 个人觉得安卓下最好用的kodi客户端是yaste, iOS下直接用"official Kodi Remote"，两者都支持手势操作.
- 在Kodi插件库搜索安装TubeCast,这样安卓和iOS下直接可以从youtube中投射节目到电视上.
- 在系统设置-服务-UPnP中打开允许UPnP控制，可以开启更多安卓应用的视频投屏支持.

