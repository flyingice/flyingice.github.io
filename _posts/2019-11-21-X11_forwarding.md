---
title: macOS X配置X11 forwarding
author: Yibin Yang
tag: ssh
---

最近看了点关于X window的东西，于是自己尝试了一下从本地ssh连接远程机器并启动GUI程序的过程。

本地机器macOS版本10.15.1，先给它装上X server。

```shell
brew cask install xquartz
```

这里XQuartz实际上同时提供了X11的服务器和客户端库。

由于家里没有两台电脑，所以先上AWS随便创建一个Linux的EC2实例，然后再从本地建立ssh远程连接：

```shell
ssh -i .ssh/EC2-default.key ec2-user@ec2-35-181-8-157.eu-west-3.compute.amazonaws.com
```

查看一下Linux发行版信息：

```shell
cat /proc/version
Linux version 4.14.152-127.182.amzn2.x86_64 (mockbuild@ip-10-0-1-129) (gcc version 7.3.1 20180712 (Red Hat 7.3.1-6) (GCC)) #1 SMP Thu Nov 14 17:32:43 UTC 2019
```

是时候给远程机器装上X客户端了：

```shell
sudo yum install xorg-x11-apps
```

这时候试着退出当前session然后带上-Y参数重新从本地建立远程ssh连接。

```shell
ssh -i .ssh/EC2-default.key -Y ec2-user@ec2-35-181-8-157.eu-west-3.compute.amazonaws.com
```

顺利登陆后试着启动xeyes，发现xeyes应用没有顺利在mac机器上跑起来。附上命令行出错信息：

```shell
[ec2-user@ip-172-31-29-110 ~]$ xeyes
Error: Can't open display:
```

退出当前session，debug模式下重新建立ssh连接。

```shell
ssh -i .ssh/EC2-default.key -Y -v ec2-user@ec2-35-181-8-157.eu-west-3.compute.amazonaws.com
debug1: Requesting X11 forwarding with authentication spoofing.
debug1: Sending environment.
debug1: Sending env LC_TERMINAL_VERSION = 3.3.7
debug1: Sending env LC_TERMINAL = iTerm2
debug1: Sending env LC_CTYPE = UTF-8
debug1: Remote: No xauth program; cannot forward with spoofing.
X11 forwarding request failed on channel 0
```

原来是远程机器上忘了安装Xauth，于是补上：

```shell
sudo yum install xorg-x11-xauth.x86_64
```

再次启动xeyes测试，大功告成。

![xeyes](https://raw.githubusercontent.com/flyingice/flyingice.github.io/master/img/xeyes.png)

X11 forwarding里不是很符合思维习惯的地方在于X服务器是在本地mac机器上，而远端AWS上运行的xeyes应用则是X客户端。X服务器在本地捕获用户的键盘及鼠标输入并转发给远程的X客户端，客户端响应输入，最终通过X服务器在本地绘制图形界面。





