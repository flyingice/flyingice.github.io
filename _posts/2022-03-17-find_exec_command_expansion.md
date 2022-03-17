---
title: Shell command expansion
author: flyingice
tag: shell
---

最近在终端下手动执行shell命令的时候发现一个小问题，大体需求是把Linux某个目录下所有文件的后缀名改掉。

为了简化实际情况，假设当前目录下有5个后缀名为x的文件，目标是把它们的后缀全部换成y，比如文件a.x改名为a.y。

```bash
> touch {a..e}.x
> la
total 0
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:41 a.x
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:41 b.x
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:41 c.x
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:41 d.x
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:41 e.x
```

最直观的方法当然是循环`for file in *; do mv $file $(basename $file .x).y; done`。这样完全没问题，但我最开始尝试的是用`find -exec`的方式:

```bash
> find . -type f -exec mv {} $(basename {} .x).y \;
> la
total 0
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:50 a.x.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:50 b.x.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:50 c.x.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:50 d.x.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 09:50 e.x.y

```

只是结果不大对，a.x没有像预期那样改名成a.y，而是变成了a.x.y。看上去命令执行后直接给每个文件都加上了.y后缀, `basename {} .x`完全没有起到作用。为了debug，我先把文件名恢复，然后尝试简单打印名字。

```bash
> find . -type f -exec basename {} .x \;	#1
e
c
a
d
b
```

`basename {} .x`按预期打印出了去掉后缀的文件名，没任何问题，再继续改改命令：

```bash
> find . -type f -exec echo $(basename {} .x) \;	#2
./e.x
./c.x
./a.x
./d.x
./b.x
```

在我的预期中命令#1和命令#2应该等价，但这下结果不对。

其实这里的问题在于`$()`命令展开(command expansion)。`$()`内的命令是在subshell中执行的，`{}`并没有被当前shell替换成真实文件名，于是subshell中运行的命令是`basename {} .x`，执行结果`{}`以string形式传回给当前shell，也就是说`-exec`真实执行的命令是`echo {}`, 导致命令#2等同于以下所示的命令#3。

```bash
> find . -type f -exec echo {} \;	#3
./e.x
./c.x
./a.x
./d.x
./b.x
```

我们真正希望的是将`{}`代表的文件名以参数形式传给`basename`，这里可以考虑借助`zsh -c`的特性:

```bash
> find . -type f -exec zsh -c 'basename "$0" .x' {} \;
e
c
a
d
b
> find . -type f -exec zsh -c 'mv $0 $(basename "$0" .x).y' {} \;
> la
total 0
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 10:21 a.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 10:21 b.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 10:21 c.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 10:21 d.y
-rw-rw-r-- 1 flyingice flyingice 0 Mar 17 10:21 e.y

```

要注意的是，`zsh -c`中`$0`不是引号中的命令本身，而代表传递给它的第一个参数，帮助文档里也提到了。

> |-c        If the -c option is present, then commands are read from the first non-option argument command_string.  If there are arguments after the command_string, the first argument is assigned to $0 and any remaining arguments are assigned to the positional parameters. The assignment to $0 sets the name of the shell, which is used in warning and error messages.

当然如果只是为了文件改名的初始需求，还可以简单运行`rename x y *.x`。不过`rename`工具可能需要在系统中手动安装。
