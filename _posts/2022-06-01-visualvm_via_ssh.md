---
title: VisualVM远程监控Java应用
author: flyingice
tag: java jvm jmx visualvm ssh
---

VisualVM是一个能够提供Java应用程序在JVM上运行时的实时状态的工具，详细信息包括CPU占用，heap分配，线程状态以及profiling数据。总体来说，它的可视化功能还是非常友好的。

如果Java应用在本地运行，打开VisualVM就可以在侧边栏看到当前活跃的Java进程，不需要任何配置，剩下就是在各个功能页面下用鼠标操作来查看各种运行时信息，[官方文档](https://visualvm.github.io/documentation.html) 对VisualVM支持的功能也有简要的说明。但是如果需要获得远程机器的JVM信息就会麻烦一些了，文档中提到了VisualVM支持远程连接，但是省略了一些关键配置。下面说下基本操作流程当作备忘。

首先根据官方文档，VisualVM的远程连接可以借助jstatd或者JMX进行。其中jstatd是JDK自带的工具，但是VisualVM只能通过它来获取远程应用的监控信息，不支持实时profiling，所以局限性很大，下面直接说明如何借助JMX来获取远程数据。[JMX](https://docs.oracle.com/en/java/javase/17/jmx/jmx-technology-architecture.html#GUID-6610F803-CB9A-439E-B675-5DD803B1978A)提供了一组管理和监测应用程序、系统和网络的API，VisualVM可以远程连接JMX agent来获取Java虚拟机的实时数据。

本地环境：Macbook M1 Pro, macOS 12.2.1 + VisualVM 2.1.3 + Amazon Corretto 17.0.2

远程环境：Macbook M1 Pro, Multipass 1.9.0 + Ubuntu 20.04 + OpenJDK 1.8.0 （本地运行Multipass虚拟机模拟远程主机）

VisualVM里可以直接通过Add JMX Connection来建立和JMX agent的连接，但是界面里不直接支持ssh的配置，所以我们需要手动建立ssh tunnel：

```bash
# ubuntu is the name of the remote host (username and ip address can be configured in ~/.ssh/config)
ssh -f -N -L 9000:localhost:9000 ubuntu
```
这样本地的9000端口会被监听，然后转发到远端的9000端口。下一步在远程主机上开启JMX对9000端口的监听：

```bash
java -Dcom.sun.management.jmxremote.port=9000 -Djava.rmi.server.hostname=localhost -Dcom.sun.management.jmxremote.rmi.port=9000 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false Application
```
Application是需要被监控的java应用，-D传入JMX的配置参数，具体细节可见[官方文档](https://docs.oracle.com/en/java/javase/17/management/monitoring-and-management-using-jmx-technology.html#GUID-805517EC-2D33-4D61-81D8-4D0FA770D1B8) 。这里*jmxremote.port*和*jmxremote.rmi.port*可以使用同一端口。这样在本地VisualVM里面建立到localhost:9000的JMX Connection之后就可以看到远端的进程了。以以下示例程序为例：

```java
import java.util.stream.LongStream;
import java.util.stream.IntStream;

public class Application {
  public static void main(String[] args) {
    System.out.println("program starts");

    int flag = 0;
    int cnt = 0;
    while (true) {
      cnt++;
      flag ^= 1;
      if(flag == 0) {
        calculate1(cnt);
      } else {
        calculate2(Math.min(cnt, 20));
      }

      try {
        Thread.sleep(1);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }

  static void calculate1(int limit) {
    long sum = IntStream.range(0, limit).sum();
    System.out.println("sum " + limit + ": " + sum);
  }

  static void calculate2(int limit) {
    long mul = LongStream.range(1, limit).reduce(1, (accumulator, n) -> accumulator * n);
    System.out.println("mul " + limit + ": " + mul);
  }
}
```

可以查看到如下监控指标：

![2022-06-01-cpu_profile](https://raw.githubusercontent.com/flyingice/flyingice.github.io/master/img/2022-06-01-cpu_profile.png)
![2022-06-01-mem_profile](https://raw.githubusercontent.com/flyingice/flyingice.github.io/master/img/2022-06-01-mem_profile.png)
![2022-06-01-monitoring](https://raw.githubusercontent.com/flyingice/flyingice.github.io/master/img/2022-06-01-monitoring.png)
![2022-06-01-threads](https://raw.githubusercontent.com/flyingice/flyingice.github.io/master/img/2022-06-01-threads.png)
