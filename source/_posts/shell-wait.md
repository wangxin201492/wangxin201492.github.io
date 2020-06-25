---
title: 【shell】我的wait为什么不能用
date: 2015-10-18 20:01:48
tags:
- cmd:wait
categories:
- shell
---

希望实现这样的一个功能：主脚本会运行几个后台进程，并希望这几个后台进程运行完之后，主进程才会退出。

RT,设想一个文件a，希望a的每一行都表示一个后台进程，有如下内容：

```
$ cat a
1
2
3
4
5
```

## 方式1

脚本`test_wait_1`这样写：

```shell
#!/bin/bash
# filename:test_wait_1

cat a | while read line
do
    echo $line &
done
wait
echo "wait done"
```

运行结果：

```
#result
$ bash test_wait_1
1
5
3
wait done
4
2
```

显然，不尽如人意。

## 方式2

脚本`test_wait_2`,做了一下修改：

```shell
#!/bin/bash
# filename:test_wait_2

while read line
do
    echo $line &
done < "a"
wait
echo "wait done"
```

运行结果：

```
$ bash test_wait_2
1
2
5
4
3
wait done
```

结果符合预期！！

## 总结

维基百科中有这样一段话：

> 其中n是当前正在执行的后台进程的pid，或工作的工作ID。如果没有给定n，命令会等待shell调用的所有工作终止。

> wait一般返回最后一个工作的退出状态。如果n所指的工作不存在，或没有工作要等待，它会返回127。

> 因为wait需要知道当前shell执行环境的工作表，它通常为shell内建命令。

这样看来仿佛`test_wait_1`中的用法也没有什么问题，但实际上shell的管道`|`实际上是产生了一级子shell，也就是在`test_wait_1`中的后台进程`echo $line &`是主进程子shell的后台进程。而`wait`只会等待当前进程的后台进程执行完毕，所以`test_wait_1`在遇到wait语句直接退出了。

而`test_wait_2`中，通过`< "a"`将a文件的内容标准输入到`while`中，并未通过管道输入到`while`中，所以后台进程属于主进程。wait会等待所有的后台进程完成以后退出，

这也就是`test_wait_1`和`test_wait_2`这2个脚本运行的不同之处。

## 参考资料：

* [https://zh.wikipedia.org/wiki/Wait_(%E5%91%BD%E4%BB%A4)](https://zh.wikipedia.org/wiki/Wait_(%E5%91%BD%E4%BB%A4))

* [http://blog.csdn.net/shuanghujushi/article/details/38186303](http://blog.csdn.net/shuanghujushi/article/details/38186303)