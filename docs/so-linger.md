# TCP `SO_LINGER` 选项对Socket.close的影响

Java中通过Socket.setSoLinger设置`SO_LINGER`选项，有三种组合形式：

- **Socket.setSoLinger(`false`, linger)**  

设置为false，这时linger值被忽略。摘自[unix network programming](http://book.douban.com/subject/1756533/)：

>The default action of close with a TCP socket is to mark the socket as closed and return to the process immediately. The socket descriptor is on longer usable by the process: it can't be used as an argument to read or write. 
TCP will try to send any data that is already queued to be sent to the other end, and after this occurs, the normal TCP connection termination sequence takes place.


如果设置为false，socket主动调用close时会立即返回，操作系统会将残留在缓冲区中的数据发送到对端，并按照正常流程关闭(交换FIN-ACK），最后连接进入`TIME_WAIT`状态。

我们可以写个演示程序，客户端发送较大的数据包后，立刻调用`close`，而server端将`Receive Buffer`设置的很小。`close`会立即返回，客户端的Java进程结束，但是当我们用`tcpdump/Wireshark`抓包会发现，操作系统正在帮你发送数据，内核缓冲区中的数据发送完毕后，发送`FIN`包关闭连接。


- **Socket.setSoLinger(`true`, `0`)**

>TCP discards any data still remaining in the socket send buffer and sends an RST to the peer, not the normal four-packet connection termination sequence.

主动调用`close`的一方也是立刻返回，但是这时TCP会丢弃发送缓冲中的数据，而且不是按照正常流程关闭连接（不发送FIN包），直接发送`RST`，对端会收到`java.net.SocketException: Connection reset`异常。同样使用tcpdump抓包可以很容易观察到。

另外有些人会用这种方式解决主动关闭放方有大量`TIME_WAIT`状态连接的问题，因为发送完`RST`后，连接立即销毁，不会停留在`TIME_WAIT`状态。一般不建议这么做，除非你有[合适的理由](http://stackoverflow.com/questions/3757289/tcp-option-so-linger-zero-when-its-required/13088864#13088864)：
>- If the a client of your server application misbehaves (times out, returns invalid data, etc.) an abortive close makes sense to avoid being stuck in CLOSE_WAIT or ending up in the TIME_WAIT state.
- If you must restart your server application which currently has thousands of client connections you might consider setting this socket option to avoid thousands of server sockets in TIME_WAIT (when calling close() from the server end) as this might prevent the server from getting available ports for new client connections after being restarted.
- On page 202 in the aforementioned book it specifically says: "There are certain circumstances which warrant using this feature to send an abortive close. One example is an RS-232 terminal server, which might hang forever in CLOSE_WAIT trying to deliver data to a stuck terminal port, but would properly reset the stuck port if it got an RST to discard the pending data."

- **Socket.setSoLinger(`true`, `linger > 0`)**

>if there is any data still remaining in the socket send buffer, the process will sleep when calling close() until either all the data is sent and acknowledged by the peer or the configured linger timer expires.
if the linger time expires before the remaining data is sent and acknowledged, close returns EWOULDBLOCK and any remaining data in the send buffer is discarded.

如果`SO_LINGER`选项生效，并且超时设置大于零，调用close的线程被阻塞，TCP会发送缓冲区中的残留数据，这时有两种可能的情况：
1. 数据发送完毕，收到对方的ACK，然后进行连接的正常关闭（交换FIN-ACK）
2. 超时，未发送完成的数据被丢弃，连接发送`RST`进行非正常关闭

类似的我们也可以构造demo观察这种场景。客户端发送较大的数据包，server端将Receive Buffer设置的很小。设置`linger`为1，调用`close`时等待1秒。注意`SO_LINGER`的单位为**秒**，好多人被坑过。假设`close`后1秒内缓冲区中的数据发送不完，使用`tcpdump/Wireshark`可以观察到客户端发送`RST`包，服务端收到`java.net.SocketException: Connection reset`异常。

最后，在使用NIO时，最好不设置`SO_LINGER`，以后会再写一篇文章分析。


