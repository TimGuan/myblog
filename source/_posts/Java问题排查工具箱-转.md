---
title: java问题排查工具箱(转)
date: 2017-04-07 11:35:30
categories: 
- 编程
- java
tags:
- java
- 问题排查
- 转载
---

# 日志相关工具
查问题的时候会非常依赖日志，因此看日志的相关工具非常重要，通常的话掌握好tail,find,fgrep,awk这几个常用工具的方法就可以，说到这个就必须说关键的异常和信息日志输出是多么的重要（看过太多异常的随意处理，例如很典型的是应用自己的ServletContextListener实现，很多的Listener实现都会变成往外抛RuntimeException，然后直接导致tomcat退出，而tomcat这个时候也不会输出这个异常信息，这种时候要查原因真的是让人很郁闷，尽管也有办法）。
日志的标准化也非常重要，日志的标准化一方面方便像我这种要查各种系统问题的人，不标准的话连日志在哪都找不到；另一方面对于分布式系统而言，如果标准化的话是很容易做日志tracing的，对问题定位会有很大帮助。
<!-- more -->

# CPU相关工具
碰到一些CPU相关的问题时，通常需要用到的工具：

* top (-H) 
top可以实时的观察cpu的指标状况，尤其是每个core的指标状况，可以更有效的来帮助解决问题，-H则有助于看是什么线程造成的CPU消耗，这对解决一些简单的耗CPU的问题会有很大帮助。
* sar 
sar有助于查看历史指标数据，除了CPU外，其他内存，磁盘，网络等等各种指标都可以查看，毕竟大部分时候问题都发生在过去，所以翻历史记录非常重要。 在我们内部，当然是用tsar，可以到分钟级，很赞。
* jstack 
jstack可以用来查看Java进程里的线程都在干什么，这通常对于应用没反应，非常慢等等场景都有不小的帮助，jstack默认只能看到Java栈，而jstack -m则可以看到线程的Java栈和native栈，但如果Java方法被编译过，则看不到（然而大部分经常访问的Java方法其实都被编译过）。
* pstack 
pstack可以用来看Java进程的native栈。
* perf 
一些简单的CPU消耗的问题靠着top -H + jstack通常能解决，复杂的话就需要借助perf这种超级利器了。 默认版本的perf呢，问题在于jit后的代码是看不到的，所以用perf会看到unknown什么的。
* cat /proc/interrupts 
之所以提这个是因为对于分布式应用而言，频繁的网络访问造成的网络中断处理消耗也是一个关键，而这个时候网卡的多队列以及均衡就非常重要了，所以如果观察到cpu的si指标不低，那么看看interrupts就有必要了。

# 内存相关工具
碰到一些内存相关的问题时，通常需要用到的工具：

* jstat 
jstat -gcutil或-gc等等有助于实时看gc的状况，不过我还是比较习惯看gc log。
* jmap 
在需要dump内存看看内存里都是什么的时候，jmap -dump可以帮助你；在需要强制执行fgc的时候（在CMS GC这种一定会产生碎片化的GC中，总是会找到这样的理由的），jmap -histo:live可以帮助你（显然，不要随便执行）。
* gcore 
相比jmap -dump，其实我更喜欢gcore，因为感觉就是更快，不过由于某些jdk版本貌似和gcore配合的不是那么好，所以那种时候还是要用jmap -dump的。
* mat 
有了内存dump后，没有分析工具的话然并卵，mat是个非常赞的工具，好用的没什么可说的。 mat的问题需要自己有大内存的机器，否则不好分析，另外就是还得把文件传来传去，所以在内部当然是用zprofiler，可以不用自己传文件，还是web版的，现在的话连机器都不用登录，更是大赞。
* btrace 
少数的问题可以mat后直接看出，而多数会需要再用btrace去动态跟踪，btrace绝对是Java中的超级神器，举个简单例子，如果要你去查下一个运行的Java应用，哪里在创建一个数组大小>1000的ArrayList，你要怎么办呢，在有btrace的情况下，那就是秒秒钟搞定的事，:)
* gperf 
Java堆内的内存消耗用上面的一些工具基本能搞定，但堆外就悲催了，目前看起来还是只有gperf还算是比较好用的一个，或者从经验上来说Direct ByteBuffer、Deflater/Inflater这些是常见问题。 除了上面的工具外，同样内存信息的记录也非常重要，就如日志一样，所以像GC日志是一定要打开的，确保在出问题后可以翻查GC日志来对照是否GC有问题，所以像-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc: 这样的参数必须是启动参数的标配。

# ClassLoader相关工具
作为Java程序员，不碰到ClassLoader问题那基本是不可能的，在排查此类问题时，最好办的还是-XX:+TraceClassLoading，或者如果知道是什么类的话，我的建议就是把所有会装载的lib目录里的jar用jar -tvf *.jar这样的方式来直接查看冲突的class，再不行的话就要呼唤btrace神器去跟踪Classloader.defineClass之类的了。

# 其他工具
* jinfo 
Java有N多的启动参数，N多的默认值，而任何文档都不一定准确，只有用jinfo -flags看到的才靠谱，甚至你还可以看看jinfo -flag，你会发现更好玩的。
* dmesg 
你的java进程突然不见了？ 也许可以试试dmesg先看看。
* systemtap 
有些问题排查到java层面是不够的，当需要trace更底层的os层面的函数调用的时候，systemtap神器就可以派上用场了。
* gdb 
更高级的玩家们，拿着core dump可以用gdb来排查更诡异的一些问题。

[原文地址](https://yq.aliyun.com/articles/61431)