---
title: "编译windows运行环境下的vim"
date: 2014-01-01 17:15:47 +0800
comments: true
tags: [ vim ]
---
整理下在windows环境中编译vim的主要步骤:
<!--more-->

- 编译gvim.exe(32/64 bits)
- 编译libiconv.dll(32/64 bits)
- 编译cscope.exe(32 bits)
- 编译diff.exe(32 bits)
- 编译ctags.exe(32 bits)

这里提供下本人编译的gvim的下载链接,包括
32位[gvim74.zip](/dl/gvim74.zip)
和64位[gvim74_x64.zip](/dl/gvim74_x64.zip),绿色版,
欢迎下载(别忘了替换自己的_vimrc和vimfiles)  
如果有兴趣可以动手照下面步骤稍微修改make参数打开更多的特性
<!--more-->
---
### 所需编译工具:
mingw64
> 因为[mingw32](http://mingw.org/)无法生成64位的exe,所以选择使用[mingw64](http://mingw-w64.sourceforge.net/), mingw64命令行在windows环境下最好运行在[cgywin](http://www.cygwin.com/)的终端环境中，这样看似互相冲突的两种linux on windows方案在编译这件事上就很完美的结合在一起

msvc(visual studio)(可选)
> 出于windows下的开发习惯可以使用visual studio自带的c/c++编译器生成gvim.exe,当然使用mingw64(gcc)已经可以做到,但也许使用微软的编译器生成代码显得更正统一些?

---
### 安装配置编译环境
- msvc: 微软C/C++编译器

获取C++编译器需要安装visual studio 2010|2012,不能使用Express版因为Express版本(2013以前)不包含生成64bits的代码工具，
但发现新发布的visual studio 2013 express中包含生成64位程序的编译器,下面步骤测试使用了2013 express证实可行
> 使用visual studio工具在命令行下编译c/c++程序的要点就是要配置以下三个环境变量
>
> - `path`:使得命令行能够找到cl.exe,mspdb*.dll,link.exe,nmake.exe,rc.exe编译链接资源构建工具等等
> - `include`:指定标准c库和平台sdk头文件路径
> - `lib`:指定和include对应的各种链接库路径
>

- mingw64: GCC

mingw64工具链在cgywin的软件仓库中有发布
cygwin的安装非常方便，运行setup程序后，
搜索安装 __make__ , __binutil__ , __openssh__  
安装编译工具链搜索*mingw64*并选中安装:
__mingw64-i686-gcc-core__ , __mingw64-i686-gcc-g++__ , __mingw64-x86_64-gcc-core__, __mingw64-x86_64-gcc-g++__  
(这里推荐网易开源镜像,速度很快)

![](/images/cygwin163.png)
![](/images/mingw64.png)

首次安装完成后启动cygwin.bat(管理员权限)并使用ssh-host-config命令配置sshd服务后即可以使用securecrt 登录ssh本机127.0.0.1的cygwin环境

---
### 开始编译:
#### =>gvim

代码在http://www.vim.org/download.php ,下载完整版: vim-7.4.tar.bz2  

__生成32 bits:__  

__使用mingw64(gcc):__  
{{< highlight bash >}}
cd src
make -f Make_ming.mak GUI=yes OLE=no MBYTE=yes ARCH=i686 USERNAME=fooy USERDOMAIN= CROSS_COMPILE=i686-w64-mingw32- 
{{< /highlight >}}
参数解释:

- `GUI=yes`: 编译图形窗口,如果指定为GUI=no,则生成dos版本
- `USERNAME=`和`USERDOMAIN=`: 当在vim中敲入`:ver`,这两个参数会显示在"Compiled by:"之后

__或者在cmd环境中使用msvc生成(可选):__  

需要先配置`include`和`lib`环境变量
{{< highlight cmd >}}
C:\Users\fy>echo %include%
C:\Program Files (x86)\Microsoft SDKs\Windows\v7.1A\Include;C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\include;
C:\Users\fy>echo %lib%
C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\lib;C:\Program Files (x86)\Microsoft SDKs\Windows\v7.1A\Lib;
{{< /highlight >}}

在`path`变量中加入路径:
{{< highlight cmd >}}
C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin;
C:\Program Files (x86)\Microsoft SDKs\Windows\v7.1A\Bin;
{{< /highlight >}}
编译:
{{< highlight bash >}}
nmake -f Make_mvc.mak GUI=yes OLE=no MBYTE=yes USERNAME=fooy USERDOMAIN= MSVCVER=12.0
{{< /highlight >}}
__生成64 bits:__  
__mingw64(gcc):__  
{{< highlight bash >}}
make -f Make_ming.mak GUI=yes OLE=no MBYTE=yes ARCH=x86-64 USERNAME=fooy USERDOMAIN= CROSS_COMPILE=x86_64-w64-mingw32-
{{< /highlight >}}

__msvc:__  
64位的代码工具和库所在目录名一般包含'amd64'或'x64',这个需要检查调整,不同版本的visual studio安装略有差异  
将`lib`调整为64位代码库路径,`include`保持不变

{{< highlight cmd >}}
C:\Users\fy>echo %lib%
C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\lib\amd64;C:\Program Files (x86)\Microsoft SDKs\Windows\v7.1A\Lib\x64; 
{{< /highlight >}}
 
`path` 也需要调整为优先定位64位的编译程序,加入路径:    
 
{{< highlight cmd >}}
C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin\x86_amd64;
C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin;
C:\Program Files (x86)\Microsoft SDKs\Windows\v7.1A\Bin\x64;
{{< /highlight >}}

编译:

{{< highlight bash >}}
nmake -f Make_mvc.mak GUI=yes OLE=no MBYTE=yes CPU=AMD64 MSVCVER=12.0 USERNAME=fooy USERDOMAIN= 
{{< /highlight >}}


以上每次编译生成的文件,只需要gvim.exe vimrun.exe xxd.exe这三个文件  
同时需要拷贝出代码目录下的runtime目录,这是vim的运行时目录(`:h $VIMRUNTIME`)

#### =>libiconv:
windows环境下缺失libiconv.dll最直接现象是vim无法正常打开`Unicode(utf-16)`和`Unicode big edian(utf-16le)`类型的文件
这里编译的32bits和64bits要和以上的vim对应放在同一目录(参加`:h iconv-dyn`)  
__32 bits:__
{{< highlight bash >}}
./configure --host=i686-w64-mingw32  && make
{{< /highlight >}}
__64 bits:__ 
{{< highlight bash >}}
./configure --host=x86_64-w64-mingw32 && make
{{< /highlight >}}
生成的文件中只需要libiconv*.dll这个库文件
{{< highlight bash >}}
 cp lib/.libs/libiconv-2.dll  libiconv.dll
{{< /highlight >}}

#### =>编译cscope
vim中编辑代码的必需套件就是cscope,移植windows下的cscope不幸有点小麻烦,需要先编译正则表达式库pcre和终端库pdcurse  
__pcre:__  
代码:http://sourceforge.net/projects/pcre/files/pcre/
{{< highlight bash >}}
./configure --host=i686-w64-mingw32 --disable-cpp && make
{{< /highlight >}}
生成库文件:`.libs/{libpcre.a,libpcreposix.a}`

__pdcurse:__  
代码:http://sourceforge.net/projects/pdcurses/files/
{{< highlight bash >}}
cd pdcurs32/win32/
make -f gccwin32.mak CC=i686-w64-mingw32-gcc LINK=i686-w64-mingw32-gcc
{{< /highlight >}}
生成库文件:`win32/pdcurses.a`

__cscope:__  
需要在cygwin中查找并安装 __flex__ 和 __bison__  
cscope的代码需要一些小的修改才可以顺利编译,这里参考了[JesseFang](http://www.cnblogs.com/JesseFang/archive/2009/08/04/1538330.html),
主要是去掉一些不存在的函数和修正小的bug
我是基于cscope-15.8a基础上修改的，代码在[这里下载](/dl/cscope-15.8a-mingw64.zip)

解压后执行configure配置编译环境,需要指定pcre和pdcurse的头文件路径
{{< highlight bash >}}
PCRE_DIR="${PWD}/../pcre-8.32/"
PDCURSE_DIR="${PWD}/../pdcurs34/"
./configure --host=i686-w64-mingw32 CFLAGS="-DPCRE_STATIC -I${PWD} -I${PCRE_DIR} -I${PDCURSE_DIR}" 
{{< /highlight >}}
对configure生成的头文件做一些处理,通过宏让编译器绕过一些错误
{{< highlight bash >}}
cat >>config.h <<!
#define __DJGPP__
#define __MSDOS__
!
{{< /highlight >}}

最后编译并静态链接前两步生成的库文件生成cscope.exe
{{< highlight bash >}}
make LDFLAGS="-L${PCRE_DIR}/.libs/" LIBS="-static -lpcreposix -lpcre ${PDCURSE_DIR}/win32/pdcurses.a /cygdrive/c/Windows/System32/kernel32.dll" 
{{< /highlight >}}

#### =>diff.exe
下载diffutils 3.3: http://ftp.gnu.org/gnu/diffutils/

应用这个[patch](/dl/diffutils-3.3.mingw.patch)  
{{< highlight bash >}}
cd diffutils-3.3
patch -p1 <../diffutils-3.3.mingw.patch
{{< /highlight >}}
编译:
{{< highlight bash >}}
./configure --host=i686-w64-mingw32  && make
{{< /highlight >}}
生成src/diff.exe

#### =>ctags.exe
下载5.8: http://ctags.sourceforge.net/

文件ctags58.zip

#### 目录结构
以上编译生成的文件放在同一目录下:
```
    |~gvim74/
    | |+runtime/      -- 从vim源代码目录拷贝过来
    | |+vimfiles/     -- 平时收集的vim扩展都放在这个目录下
    | |-_vimrc
    | |-cscope.exe
    | |-ctags.exe
    | |-diff.exe
    | |-gvim.exe
    | |-libiconv.dll 
    | |-regvim.bat    -- 注册edit with Vim
    | |-vimrun.exe
    | |-xxd.exe
```
打开gvim ,执行`:helptags $VIM/vimfiles/doc`更新下所有的插件的帮助文档tag  
最后把这个目录打包，这样自己的vim绿色版就算制作完工  

对了这里regvim.bat这个批处理可以把vim注册到右键菜单'edit with Vim'和'文件属性->打开方式'

{{< highlight cmd >}}
@echo off
set FDIR=%~dp0
set GVIM_PATH=%FDIR%gvim.exe
IF NOT EXIST "%GVIM_PATH%" goto NOTFOUND

@echo on

@echo "@> add to right mouse file menu"

reg add "HKCR\*\shell\edit with Vim"  /f
reg add "HKCR\*\shell\edit with Vim" /d "edit with &Vim" /f
reg add "HKCR\*\shell\edit with Vim\command" /d "\"%GVIM_PATH%\"  --remote-tab-silent \"%%1\"" /f

@echo "@> add to file property - open with"
reg add "HKCR\Applications\gvim.exe\shell\open\command" /d "\"%GVIM_PATH%\"  --remote-tab-silent \"%%1\"" /f

@echo off
GOTO END

:NOTFOUND
@echo "pls put this file the same dir with gvim.exe"

:END

pause
{{< /highlight >}}
