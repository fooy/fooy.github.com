---
title: "使用CoreElec的RAID1实践"
date: 2020-06-30 00:00:02 +0800
comments: true
tags: [ CoreElec,linux ]
---

<!--more-->
最近打算使用N1+CoreElec做长期网盘，我有两块10年左右的硬盘闲置许久，适合接上N1做存储，这样也算是和N1百元盒子物尽其用的精神十分契合.
但这两块硬盘出厂确实有些年头, 考虑到数据面临的"崩盘"风险，正好可以利用这个机会折腾下Linux的Soft RAID.

N1盒子上仅有两个USB接口, 查资料简单评估觉得适合做RAID1, 能够承担一个硬盘故障的风险，看到网络上对RAID的可靠性有一定质疑,
以及对[bitrot](https://serverfault.com/questions/77710/is-bit-rot-on-hard-drives-a-real-problem-what-can-be-done-about-it)的讨论，
总体感觉个人简单使用还是能提供一定的保护的,决定实践一把让这两块硬盘接收时间的检验.

#### 编译:

CoreElec的编译过程在[上篇](../n1-coreelec-install/)有所提及，这里只列出RAID相关的编译配置部分.

1. 修改内核编译配置加入raid0,1,4,5,6支持   
projects/Amlogic/linux/linux.aarch64.conf
{{< highlight diff >}}
+CONFIG_BLK_DEV_MD=m
+CONFIG_MD_LINEAR=m
+CONFIG_MD_RAID0=m
+CONFIG_MD_RAID1=m
+CONFIG_MD_RAID10=m
+CONFIG_MD_RAID456=m
+CONFIG_DM_RAID=m
+CONFIG_ASYNC_CORE=m
+CONFIG_ASYNC_MEMCPY=m
+CONFIG_ASYNC_XOR=m
+CONFIG_ASYNC_PQ=m
+CONFIG_ASYNC_RAID6_RECOV=m
{{< /highlight >}}

2. 加入mdadm编译安装包文件，这里使用4.0版本  
packages/sysutils/mdadm/package.mk
{{< highlight diff >}}
+PKG_NAME="mdadm"
+PKG_VERSION="4.0"
+PKG_ARCH="any"
+PKG_LICENSE="LGPLv2"
+PKG_SITE="http://neil.brown.name/blog/mdadm"
+PKG_URL="http://www.kernel.org/pub/linux/utils/raid/$PKG_NAME/$PKG_NAME-$PKG_VERSION.tar.gz"
+PKG_DEPENDS_TARGET="toolchain systemd"
+PKG_SECTION="system"
+PKG_SHORTDESC="mdadm: manager for raid arrays"
+PKG_LONGDESC="mdadm is a tool for managing Linux Software RAID arrays."
+PKG_IS_ADDON="no"
+PKG_AUTORECONF="no"
+
+pre_configure_target() {
+  export CFLAGS+="  -Wno-implicit-fallthrough"
+  STRIP="--strip --strip-program=$STRIP"
+  PKG_MAKE_OPTS_TARGET="SYSCONFDIR=/storage/.config CC=$CC CWFLAGS="
+  PKG_MAKEINSTALL_OPTS_TARGET="SYSTEMD_DIR=/usr/lib/systemd/system BINDIR=/usr/sbin install-systemd"
+}
{{< /highlight >}}
3. 加入busybox依赖安装到文件系统   
packages/sysutils/busybox/package.mk
{{< highlight diff >}}
-PKG_DEPENDS_TARGET="toolchain busybox:host hdparm dosfstools e2fsprogs zip unzip usbutils parted procps-ng gptfdisk libtirpc"
+PKG_DEPENDS_TARGET="toolchain busybox:host hdparm dosfstools e2fsprogs zip unzip usbutils parted procps-ng gptfdisk libtirpc mdadm"
{{< /highlight >}}

重新编译系统生成的img文件中已经包含所需的模块.


#### CoreElec配置:

- 设置raid10模块在系统启动时载入，如果上一步raid相关内核配置为Y这一步可以省去
{{< highlight text >}}
CoreELEC:~/.config/modules-load.d # echo raid10 > raid.conf
{{< /highlight >}}

- 查看mdadm的所有安装文件，可以看到mdadm附带了很多有用的自动配置文件，包括处理热插拔相关的udev规则，和udev规则配合的systemd任务
    比如: 
    - mdmonitor.service 启动常驻进程监控raid的健康情况并在异常发生后发出邮件  
    - mdadm-last-resort@.service 在磁盘故障时也能加载部分可用的磁盘  

{{< highlight shell >}}
root@776f0f3b8d0f:/app/CoreELEC/build.CoreELEC-Phicomm-N1.arm-9.2.1/mdadm-4.0/.install_pkg# find . -type f
./usr/sbin/mdadm
./usr/sbin/mdmon
./usr/lib/systemd/system/mdadm-grow-continue@.service
./usr/lib/systemd/system/mdmon@.service
./usr/lib/systemd/system/mdadm-last-resort@.timer
./usr/lib/systemd/system/mdmonitor.service
./usr/lib/systemd/system/mdadm-last-resort@.service
./usr/lib/systemd/system-shutdown/mdadm.shutdown
./usr/lib/udev/rules.d/64-md-raid-assembly.rules
./usr/lib/udev/rules.d/63-md-raid-arrays.rules
{{< /highlight >}}


- mdadm进程默认的监控功能是调用/usr/bin/sendmail来发送邮件，Coreelec文件系统中并不包含这个古老的命令路径, 可以使用兼容的替代`msmtp`,
从entware安装:

{{< highlight text >}}
CoreELEC:~/.config/system.d # opkg install msmtp
Package msmtp (1.8.10-1) installed in root is up to date.
{{< /highlight >}}

- 修改mdadm4.0代码中某处的宏定义把`/usr/bin/sendmail`改为`/opt/bin/msmtp`后，把编译重新生成的mdadm拷贝到盒子上, 这里是利用CoreElec(LibReelec)支持用户目录自定义配置(systemd,udev,etc)覆盖系统只读路径下的同名配置的功能，在自定义配置中使用修改过邮件发送命令路径的`/storage/bin/mdadm` 同时加入邮件地址参数选项`--mail`, 这样mdmonitor任务就可以把故障邮件发送到自定义的邮箱:

{{< highlight diff >}}
CoreELEC:~/.config/system.d # diff mdmonitor.service /usr/lib/systemd/system/mdmonitor.service
19c16
< ExecStart=/storage/bin/mdadm --monitor $MDADM_MONITOR_ARGS --mail ??@qq.com
---
> ExecStart=/usr/sbin/mdadm --monitor $MDADM_MONITOR_ARGS
{{< /highlight >}}
覆盖的systemd任务重新装载即可:
{{< highlight bash >}}
systemctl daemon-reload
#udevadm trigger
systemctl restart mdmonitor.service
{{< /highlight >}}

#### 测试:
- 创建raid1(far2):
{{< highlight text >}}
CoreELEC:~ # ./mdadm --create --verbose --level=10 --metadata=1.2 --chunk=512 --raid-devices=2 --layout=f2 /dev/md/MyRAID10Array /dev/sda1 /dev/sdb1
CoreELEC:~ # cat /proc/mdstat
Personalities : [raid0] [raid1] [raid10] [raid6] [raid5] [raid4]
md127 : active raid10 sdb1[1] sda1[0]
      312438784 blocks super 1.2 512K chunks 2 far-copies [2/2] [UU]
      [>....................]  resync =  0.0% (258432/312438784) finish=382.4min speed=13601K/sec
      bitmap: 3/3 pages [12KB], 65536KB chunk

unused devices: <none>
{{< /highlight >}}
- 创建文件系统并挂载:
{{< highlight text >}}
CoreELEC:~ # mkfs.ext4 -v -L array -E stride=64,stripe-width=128 /dev/md127
CoreELEC:~ # udevil mount /dev/md127
Mounted /dev/md127 at /media/array
{{< /highlight >}}

- 重新装配创建过的阵列(上面提到的udev规则会在系统启停或者热插拔条件下自动处理):
{{< highlight txt >}}
CoreELEC:~ # ./mdadm --assemble  /dev/md123 /dev/sda1 /dev/sdb1
mdadm: /dev/md123 has been started with 2 drives.
{{< /highlight >}}
- 测试邮件:
{{< highlight bash >}}
mdadm --monitor --scan --test -1 --mail fooy84@gmail.com
{{< /highlight >}}
- 模拟故障:  
   - 拔除任意一个挂在usb口的硬盘，通过samba继续网络读写文件,并没有什么问题, 同时配置的邮箱里收到警告邮件: DegradedArray Event on /dev/md127:....
   - 这个拔除的硬盘可以用来单独启动一个raid设备,并且通过添加另外一个硬盘来继续复制.
{{< highlight txt >}}
CoreELEC:~ # ./mdadm --assemble  /dev/md123 /dev/sda1
mdadm: /dev/md123 assembled from 1 drive - need all 2 to start it (use --run to insist).
CoreELEC:~ # ./mdadm --assemble  /dev/md123 /dev/sda1 --run
mdadm: /dev/md123 has been started with 1 drive (out of 2).
{{< /highlight >}}
   - 把故障(拔除)的硬盘重新加入到原来的raid，需要先停止当前raid设备,然后使用`--add`加入
{{< highlight txt >}}
CoreELEC:~ # udevil unmount /dev/md123
udevil: success running umount as current user
CoreELEC:~ # ./mdadm --stop /dev/md123
mdadm: stopped /dev/md123
CoreELEC:~ # ./mdadm --assemble  /dev/md123 /dev/sdb1 /dev/sda1
mdadm: /dev/md123 has been started with 1 drive (out of 2).
CoreELEC:~ # ./mdadm /dev/md123 --add /dev/sdb1
mdadm: re-added /dev/sdb1
CoreELEC:~ # cat /proc/mdstat
Personalities : [raid0] [raid1] [raid10] [raid6] [raid5] [raid4]
md123 : active raid10 sdb1[2] sda1[1]
      312438784 blocks super 1.2 512K chunks 2 far-copies [2/1] [_U]
      [====>................]  recovery = 22.4% (70254656/312438784) finish=0.1min speed=35127328K/sec
      bitmap: 2/3 pages [8KB], 65536KB chunk

unused devices: <none>
{{< /highlight >}}
    - 实验可以看到，重新装配传入多个设备时，总是以未发生故障(被拔出)的分区来创建raid设备，同时需要显式的添加另一块数据可能不一致的分区来进行同步恢复,这样的设计也是合理。

    - 尝试用供电的USB hub接入第三块磁盘并添加到raid设备, 磁盘显示为Spare.
    - 最后还有一个推荐的数据保养功能，在cron中做一个[scrubbing](https://wiki.archlinux.org/index.php/LVM_on_software_RAID#Scrubbing)任务也很有必要.

#### 参考:

[raid10](https://en.wikipedia.org/wiki/Non-standard_RAID_levels#Linux_MD_RAID_10)   
[bitrot](https://serverfault.com/questions/77710/is-bit-rot-on-hard-drives-a-real-problem-what-can-be-done-about-it)  
[LibreELEC RAID support](https://www.artembutusov.com/libreelec-raid-support/)  
[A_guide_to_mdadm](https://raid.wiki.kernel.org/index.php/A_guide_to_mdadm)  
