# Linux内核参数的配置方法

`/proc`是一个伪文件系统，可以像访问普通文件系统一样访问系统内部的数据结构，获取当前运行的进程、统计和硬件等各种信息。例如可以使用`cat /proc/cpuinfo`获取CPU信息。

`/proc/sys/`下的文件和子目录比较特别，它们对应的是系统内核参数，更改文件内容就意味着修改了相应的内核参数，可以简单的使用`echo`命令来完成修改：

```
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
```

上面这个命令启用了`TCP SYN Cookie`保护。使用`echo`修改内核参数很方便，但是系统重启后这些修改都会消失，而且不方便配置参数的集中管理。`/sbin/sysctl`命令就是用来查看和修改内核参数的工具。`sysctl -a`会列出所有内核参数当前的配置信息，比遍历目录`/proc/sys/`方便多了。`sysctl -w`修改单个参数的配置，例如：

```
sysctl -w net.ipv4.tcp_syncookies=1
```
    
和上面`echo`命令的效果一样。需要注意的是，要把目录分隔符斜杠`/`替换为点`.`，并省略`proc.sys`部分。

通过`sysctl -w`修改，还是没有解决重启后修改失效的问题。更常用的方式是，把需要修改的配置集中放在`/etc/sysctl.conf`文件中，使用`sysctl -p`重新加载配置使其生效。在系统启动阶段，`init`程序会运行`/etc/rc.d/rc.sysinit`脚本，其中包含了执行`sysctl`命令，并使用了`/etc/sysctl.conf`中的配置信息。因此放在`/etc/sysctl.conf`中的系统参数设置在重启后也同样生效，同时也便于集中管理修改过了哪些内核参数。

最后，哪里有比较完整的内核参数说明文档？我觉得[kernel.org](https://www.kernel.org/doc/Documentation/sysctl/ )的文档比较全。例如我们常会遇到的网络内核参数，[net.core](https://www.kernel.org/doc/Documentation/sysctl/net.txt) 和 [net.ipv4]( https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) 。TCP相关的参数，也可以通过[man文档](http://man7.org/linux/man-pages/man7/tcp.7.html)了解。

