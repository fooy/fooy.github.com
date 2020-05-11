---
layout: post
title: "使用bison实现简单的表达式"
date: 2015-06-16 02:31:48 +0800
comments: true
tag: [bison, compiler,c++]
---

在这家公司混日子这么久，最近不幸被领导抓壮丁拉来做一个项目，其中一块功能需要表达式解析功能
<!--more-->
本来这块功能是有现成的代码可以复用，可惜原有的.y文件已经丢失, 仅存的.c文件被一代代码农改的面目全非，实在是看不下去，就翻了下bison和flex的文档，试着重新实现了一把,这是一种很简单形式的表达式，没有复杂的语法规则，主要的语法实现部分的代码量也就大概400行左右,基本还是满足需求的.

编译原理是门深奥的学问，我很惭愧毕业这么多年至今面对DFA,NFA,LR(K),LALR,SLR等等那么一坨坨的计算理论概念绕不过弯(再次拜服knuth大神)，下面只是列出从实用角度我认为使用bison编写语法需要知道的一些知识点

- 首先要明白bison处理算法是自下而上的规约   
    不需要具体了解LR文法的规约算法细节，但还是要大概明白top-down和down-top解析的区别,这样大概对每条语法规则规约动作的执行顺序就心里有数了
- 文法设计  
    搜索借鉴下网络上现成的其它语言的文法会很有帮助
- 参考手册  
    - 参考手册中提到的一些规则点还是要注意的，比如:
    递归的语法规则一定要写为[左递归](https://www.gnu.org/software/bison/manual/html_node/Recursion.html#Recursion)
    - [mid-rule actions](https://www.gnu.org/software/bison/manual/html_node/Using-Mid_002dRule-Actions.html#Using-Mid_002dRule-Actions)

### 表达式功能介绍
下面就是这个表达式语法实现的功能，没什么新奇的特性，所以适合放在这里做demo  
- 基本数据类型: int,float,string,null,byte  
- 基本的加减乘除和一些逻辑运算  
- 内置函数调用fuc(arg1,...argn)  
- 条件表达式操作符?:  

### 关于代码  

[项目地址](https://github.com/fooy/texpr)

- bison支持生成C++文件，语法实现用C++也能省去了不少代码量
- flex也可以生成C++,可惜产生的文件依赖于flex自带的头文件，为了尽量减少依赖就按照[flex文档](http://flex.sourceforge.net/manual/Cxx.html)说明生成的C代码直接用C++编译就好了


### 测试  

编译生成执行文件texpr，执行接受一个表达式字符串参数,如:
{{< highlight bash >}}

$ ./texpr 'int1'
ret : 123
$ ./texpr 'int1 + 2.1 == 125.1 && 5.1/2 == 2.5'
ret : FALSE
$ ./texpr '2 * int1 / 2.0 '
ret : 123.000000
$ ./texpr 'print( str2 == "456")'
TRUE
ret : 1
$ ./texpr 'int1 + str2 != "123456"? "equal" : STRLEN(str2)==0?3*7: "return" '
ret : "return"
$ ./texpr '1 + 2 * 3 & 8'
ret : 0
$ ./texpr 'TRIM("  " + this.is.array[1] ) + " ok"+ ( STRLEN("hello")+10 )'
ret : "789 ok15"

{{< /highlight >}}
