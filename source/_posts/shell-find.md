---
title: 【shell】再看一眼find--find使用中遇到的问题分析
date: 2015-08-15 20:01:48
tags:
- cmd:find
categories:
- shell
---

## 1 简单find命令

### 1.1 目录结构

```bash
代码1
$ tree
.
|-- 2.log
`-- backup
    `-- 1.log

3 directories, 0 files
```

### 1.2 简单的find命令

```bash
代码2
$ find -path ./backup
./backup
```

### 1.3 错误的通配符使用

```bash
代码3
$ mkdir backup123
$ find -path ./backup*
find: paths must precede expression
Usage: find [path...] [expression]
```

`find`命令报错：路径必须先表达。问题分析（引自：[http://www.cnblogs.com/baibaluo/archive/2012/08/16/2642403.html](http://www.cnblogs.com/baibaluo/archive/2012/08/16/2642403.html)）:

当目录下存在多个`backup*`，shell命令变成`find -path backup backup123`.此时，`-name`后面有2个匹配字符，shell报错。

**解决办法**：-path(-name)匹配的字符串已经要用单引号，或者双引号引住。

### 1.4 正确的通配符写法

```bash
代码4
$ find -path './backup*'
./backup
./backup/1.log
./backup123
```

此时，命令运行正确！

## 2 find多条件

`expr1 expr2 -o expr3` 等同于 `expr1 -a expr2 -o expr3`.与其他语言中的**与或非**类似。并且是**短路求值**

### 2.1 find多条件尝试:与

	代码5
	$ find -path ./backup -name '*.log'
	$

没有任何返回结果。该语句的含义是：路径是path，并且名字是以.log结尾的文件。显然，并不存在。

本语句实际上是想查找，backup下所有的以.log结尾的文件。应是：

1. `find -path './backup*' -name '*.log'`

### 2.2 find多条件尝试:或

	代码6
	$ find -path './backup*' -o -name '*.log'
	./backup
	./backup/1.log
	./2.log
	./backup123

到这个地方，`-a` 和 `-o`的体会已经一目了然了吧。这条命令展示了在backup*下的所有文件和以.log结尾的所有文件。

## 3 `-prune`的体会

贴上这样的几条`shell`命令，请先自行体会：

	代码7
	$ find -path './backup*'
	./backup
	./backup/1.log
	./backup123
	$ find -path './backup*' -prune
	./backup
	./backup123

### 3.1 `prune`的基本使用

`-prune`在man中是这么说的

> If -depth is not given, true; do not descend the current directory.         
> If -depth is given, false; no effect.

如果`find`语句中存在-depth选项，那么`-prune`将会被忽略。否则，`-prune`将声明不展开当前路径。

这样在上述的1、2条命令中，由于`-prune`选项的存在，致使backup路径没有展开。所以1.log没有在打印列表中。

我们再次做这样的尝试：

	代码8
	$ touch backup123/3.log
	$ find -path './backup*' -prune
	./backup
	./backup123
	$ find -path './backup*'
	./backup
	./backup/1.log
	./backup123
	./backup123/3.log

打印的结果和预期是一样的。

按照上述**2 find多条件**中说道的那样，`find -path './backup*'`获得所有backup前缀的文件，然后将结果和`-prune`相`与`：其实就是判断前者的结果中是否包含**指定路径**的子文件(夹)。

### 3.2 `prune`做排除路径用

而一般情况下`prune`是这样使用的

	代码9
	$ find -path './backup*' -prune -o -name '*.log' -print
	./2.log

指代的意思是当前路径除去`backup*`文件夹外的所有`*.log`文件。

#### 3.2.1 一个问题

这样是如愿以偿了，但是我们执行一下这样的一条命令：

	代码10
	$ find -path './backup*' -o -name '*.log' -print
	./2.log

返回的结果一模一样。

#### 3.2.2 进一步剖析

这个问题我们暂且搁置不论，继续来看这样的2个命令：

	代码11
	$ find -path './backup*' -prune -o -name '*.log'
	./backup
	./2.log
	./backup123
	$ find -path './backup*'  -o -name '*.log'
	./backup
	./backup/1.log
	./2.log
	./backup123

2个结果集中只是缺少了`./backup/1.log`，`-prune`做到的只是一个收缩路径的功能。

再继续对比这2个命令和上面两个命令，缺少的是一个`-print`.其实在man里面有这样的一句话**"If no expression is given, the expression '-print' is used."**

也就是说`-print`是个默认值，那么上面2组命令实际上可以这样看待：

|                     输入命令                     |                           实际命令                           |
| :----------------------------------------------: | :----------------------------------------------------------: |
|    `find -path './backup*' -o -name '*.log'`     |    `find \( -path './backup*' -o -name '*.log' \) -print`    |
| `find -path './backup*' -o -name '*.log' -print` | `find \( -path './backup*' \) -o \( -name '*.log' -print \) ` |

**代码9** 和 **代码10**中的片段可以理解为打印`-path './backup*'`为false 、 `-name '*.log'` 为true的find结果。

而**代码11**中的片段则是将`-path './backup*' -o -name '*.log'`过滤后所有为true的结果都打印。

#### 3.2.3 总结

那么这样看来，其实排除路径其实是将`-print`放置到了-o后面作为输出。而`-path './backup*'`执行过，并且返回true，单并未被打印。

那么是不是说，其实，其实，其实`-prune`并没有什么用？

## 4 总结：多一点角度看find

其实可以认为

1. find无可避免的对指定路径进行了全文搜索，默认情况下是深度优先搜索(只有在指定-depth的时候使用广度优先搜索)。
2. find进行全文搜索以后，将结果扔到后面的过滤条件中，按照**与或非**的规则，逐条过滤。
3. 最终返回值为true的item被打印了出来


试想这样一个场景，在一个java项目中，由于项目庞大，总文件数上万。想要找到最深2级目录下所有的java文件。

	find . -name "*.java" -maxdepth 2
	find . -maxdepth 2 -name "*.java"

这样的2条命令，显然第二条的执行效率会快！！！`-maxdepth 2` 极大程度的进行了一次结果过滤。

那么在写find命令的时候，应该把能最大程度减小结果集的结果放到前面。