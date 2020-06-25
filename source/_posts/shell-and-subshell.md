---
title: 【shell】shell与子shell那团乱麻
date: 2015-12-16 20:01:48
tags:
- cmd:fork
- cmd:source
- cmd:exec
categories:
- shell
---


## 1. `fork` & `source` & `exec`

在运行shell脚本时候，有三种方式来调用外部的脚本，`exec(exec script.sh)`、`source(source script.sh)`、`fork(./script.sh)`

### 1.1. exec（exec /home/script.sh）：

使用`exec`来调用脚本，被执行的脚本会继承当前shell的环境变量。**但事实上`exec`产生了新的进程，他会把主shell的进程资源占用并替换脚本内容，继承了原主shell的PID号，即原主shell剩下的内容不会执行。**

### 1.2. source（source /home/script.sh）

使用`source`或者“`.`”来调用外部脚本，**不会产生新的进程**，继承当前shell环境变量，而且被调用的脚本运行结束后，**它拥有的环境变量和声明变量会被当前shell保留**，类似将调用脚本的内容复制过来直接执行。**执行完毕后原主shell继续运行。**

### 1.3. fork（/home/script.sh）

直接运行脚本，会**以当前shell为父进程，产生新的进程**，并且**继承主脚本的环境变量和声明变量**。执行完毕后，**主脚本不会保留其环境变量和声明变量**。

总结：这样来看`fork`最灵活，`source`次之，`exec`最诡异。

### 1.4. example

##### 主脚本

```shell
#!/bin/sh
a=main

echo "a is $a"
echo "PID for parent before 2.sh:$$"
case $1 in
  exec)
    echo "using exec"
    exec ./2.sh ;;
  source)
    echo "using sourcing"
    source ./2.sh ;;
  *)
    echo "using fork"
    ./2.sh ;;
esac

echo "PID FOR parent after 2.sh :$$"

echo "now main.sh a is $a"
echo "$b"
```

##### 调用脚本：2.sh

```shell
#!/bin/sh
echo "PID FOR 2.SH:$$"

echo  "2.sh get a from main.sh is $a"

a=2.sh
export a
b=3.sh

echo "now 2.sh a is $a"
```

##### 执行结果：

```shell
$ ./main.sh exec
a is main
PID for parent before 2.sh:19026
using exec
PID FOR 2.SH:19026
2.sh get a from main.sh is main
now a is 2.sh
```

```shell
$ ./main.sh source
a is main
PID for parent before 2.sh:19027
using sourcing
PID FOR 2.SH:19027
2.sh get a from main.sh is main
now a is 2.sh
PID FOR parent after 2.sh :19027
now main.sh a is 2.sh
3.sh
```

```shell
$ ./main.sh fork
a is main
PID for parent before 2.sh:19028
using fork
PID FOR 2.SH:19029
2.sh get a from main.sh is main
now a is 2.sh
PID FOR parent after 2.sh :19028
now main.sh a is main
```

## 参考资料

1. [http://qujunorz.blog.51cto.com/6378776/1541676](http://qujunorz.blog.51cto.com/6378776/1541676)