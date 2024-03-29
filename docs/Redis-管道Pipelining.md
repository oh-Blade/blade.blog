# 概念
Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。
这意味着通常情况下一个请求会遵循以下步骤：
* 客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
* 服务端处理命令，并将结果返回给客户端。

客户端和服务器通过网络进行连接。这个连接可以很快（loopback接口）或很慢（建立了一个多次跳转的网络连接）。无论网络延如何延时，数据包总是能从客户端到达服务器，并从服务器返回数据回复客户端。这个时间被称之为 RTT (Round Trip Time - 往返时间). 

Redis提供了批处理操作命令（mget、mset），但大部分命令不支持批量操作。Pipline提供了一种解决方案。可以将一组Redis命令进行组装，通过一次RTT传输给Redis，再将执行结果按顺序返回给客户端。

RTT再不同网络环境下会有不用，同机房或同机器会比较快，跨机房跨地区会比较慢，Redis命令真正执行的时间通常会在微妙级别，所以才会有Redis性能瓶颈是网络这样的说法。

redis-cli的--pipe就是使用的Pipeline机制

# Jedis中使用
利用jedis对象生成一个pipeline对象，直接可以调用jedis.pipelined().
然后将要执行的命令封装到pileline中，此时不会真正执行命令。
最后调用pileline.sync()完成调用。
