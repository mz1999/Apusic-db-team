# 使用USE Method分析系统性能问题

遇到性能问题怎么分析定位？这个问题太难回答了，各种底层环境、依赖系统、业务场景，怎么可能有统一的答案。于是产生了各种分析性能问题的“流派”。两个典型的 ANTI-METHODOLOGIES：

- **blame-someone-else**
使用此方法的人遵循下列步骤：
    1. 找到一个不是他负责的系统或环境
    2. 假定问题和这个组件有关
    3. 将问题转交个负责这个组件的团队
    4. 如果证明是错误的，重复步骤1


- **路灯法**
没有系统的方法论，只是使用自己擅长的工具去观察，而不管问题到底出现在哪儿。就像丢了钥匙的人去路灯下寻找，仅仅是因为路灯下比较亮。这种行为被称为[路灯效应](http://en.wikipedia.org/wiki/Streetlight_effect)。

相信很多同学已经脑补出上述的两个场景，他们的行为模式让人抓狂。于是有聪明人总结出了《[The USE Method](http://www.brendangregg.com/usemethod.html)》。USE是Utilization，Saturation 和 Errors的缩写，简单说USE是一套分析系统性能问题的方法论，具体表现为一个checklist，分析过程就是对照checklist一项项检查，希望能快速定位瓶颈资源或错误。

初看这个方法感觉有点太简单了吧，这也能称为方法论？不过这确实体现出了老外的做事风格，任何事情都会去做定量分析，力求逻辑完整。而我们往往讳莫高深的一笑，只可意会不可言传。

简单介绍下USE，详细内容推荐看这篇《[The USE Method](http://www.brendangregg.com/usemethod.html)》。USE的一句话总结：
>For every resource, check utilization, saturation, and errors.

术语解释
- **resource**：CPU，内存，磁盘，网络等一切物理设备资源
- **utilization**：资源利用率。例如CPU的资源利用率90%
- **saturation**：当资源繁忙时仍能接收新的任务，这些额外的任务一般都放入了等待队列。`saturation`就表现为队列的长度，例如CPU的平均运行队列为4（Linux上使用vmstat命令获得）。
- **errors**：系统的错误报告数，例如`TCP`监听队列`overflowed`次数。

列出系统中的所有资源，然后逐项检查利用率、等待队列和错误数，就这么简单！下表是一个范例：

| resource      |     type |   metric   |
| :-------- | --------:| :------: |
|CPU|	utilization	|CPU utilization (either per-CPU or a system-wide average)|
|CPU|	saturation	|run-queue length |
|Memory capacity|	utilization	|available free memory (system-wide)|
|Memory capacity	|saturation	|anonymous paging or thread swapping|
|Network interface	|utilization	|RX/TX throughput / max bandwidth Storage|
|Storage device I/O	|utilization	|device busy percent|
|Storage device I/O	|saturation	|wait queue length|
|Storage device I/O	|errors	|device errors ("soft", "hard", ...)|

对于资源测量数据的解读，作者给了一些建议，例如：资源利用率100%肯定表示该资源是系统瓶颈，70%以上的利用率就要引起足够的重视，一般IO设备利用率高于70%，响应时间将大幅上升。资源等待队列大于0意味着可能存在问题。资源的任何错误计数，都值得仔细调查，特别是当性能变差时，错误计数在上升。

要使用这个方法，你还需要一份完整的资源列表，一般的系统资源包括：

- CPUs: sockets, cores, hardware threads (virtual CPUs)
- Memory: capacity
- Network interfaces
- Storage devices: I/O, capacity
- Controllers: storage, network cards
- Interconnects: CPUs, memory, I/O

作者很厚道的按照每种操作系统给出了checklist，重点关注《[USE Method: Linux Performance Checklist](http://www.brendangregg.com/USEmethod/use-linux.html)》，不仅列出了资源，而且告诉你如何进行测量。例如CPU运行队列的测量：
> system-wide: vmstat 1, "r" > CPU count [2]; sar -q, "runq-sz" > CPU count; dstat -p, "run" > CPU count; per-process: /proc/PID/schedstat 2nd field (sched_info.run_delay); perf sched latency (shows "Average" and "Maximum" delay per-schedule); dynamic tracing, eg, SystemTap schedtimes.stp "queued(us)"

根据作者的实践经验，使用USE方法解决了80%的性能问题，只付出了5%的努力，当考虑了所有的资源，你不太可能忽视任何问题。简单有效！


