---
layout: post
title:  "从 SYN Drop 看 tomcat 线程模型"
date:   2017-05-11
categories: tomcat
---

# 问题背景

近期在业务高峰期, 发现会出现客户端请求发送到服务端无法响应的情况. 通过查看机器监控, 我们发现出现了很多 syn drop 的情况.

# SYN Drop

要想查找问题根源所在, 首先需要理解 syn drop 的含义和成因. 

在 tcp 的连接过程中, 会用到一个叫 *BacklogQueue* 的队列, 这个队列用于存储那些三次握手成功后, 处于*等待accept*状态的 socket, accept queue 的大小由操作系统参数 backlog 决定. 每次 server 端应用程序调用 socket.accept(), 就会从 BacklogQueue 获取一个 socket. 一旦这个队列满了以后, 就会出现 syn drop 的情况, 即三次握手的 syn 包被直接丢弃, 不给客户端任何回复, 这样就会导致客户端出现连接无法建立的情况. 

# 应用的问题

不难发现, 出现 syn drop 的原因是因为应用没有及时的 accept socket, 导致 BacklogQueue 的堆积. 对应到我们的服务端应用, 实际上就是 tomcat 对连接的处理程序. 我们的 tomcat 版本是 7.0.47, 基本都是使用的 bio acceptor, 下面看 bio 的线程模型.

![](/images/2016-11-04-welcome-to-jekyll/89af2252.png)

tomcat 有一个专门的 Acceptor (参见 JioEndpoint.Acceptor) 的用来 accept socket, 它是一个不停的死循环线程, 不断的 accept socket, 并将 socket 派发给线程池去处理实际的数据. 结合实际的 acceptor 代码实现

````java
    protected class Acceptor extends AbstractEndpoint.Acceptor {

        @Override
        public void run() {

            int errorDelay = 0;

            // Loop until we receive a shutdown command
            while (running) {

                // Loop if endpoint is paused
                while (paused && running) {
                    state = AcceptorState.PAUSED;
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }

                if (!running) {
                    break;
                }
                state = AcceptorState.RUNNING;

                try {
                    //if we have reached max connections, wait
                    countUpOrAwaitConnection();

                    Socket socket = null;
                    try {
                        // Accept the next incoming connection from the server
                        // socket
                        socket = serverSocketFactory.acceptSocket(serverSocket);
                    } catch (IOException ioe) {
                        countDownConnection();
                        // Introduce delay if necessary
                        errorDelay = handleExceptionWithDelay(errorDelay);
                        // re-throw
                        throw ioe;
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // Configure the socket
                    if (running && !paused && setSocketOptions(socket)) {
                        // Hand this socket off to an appropriate processor
                        if (!processSocket(socket)) {
                            countDownConnection();
                            // Close socket right away
                            closeSocket(socket);
                        }
                    } else {
                        countDownConnection();
                        // Close socket right away
                        closeSocket(socket);
                    }
                } catch (IOException x) {
                    if (running) {
                        log.error(sm.getString("endpoint.accept.fail"), x);
                    }
                } catch (NullPointerException npe) {
                    if (running) {
                        log.error(sm.getString("endpoint.accept.fail"), npe);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("endpoint.accept.fail"), t);
                }
            }
            state = AcceptorState.ENDED;
        }
    }
````

从代码上看, 这个模型相对来说还是比较简单明的, 但这里会涉及到几个关键参数

* maxConnections: tomcat 最大并发连接数, 默认与 maxThreads 相同
* maxThreads: tomcat 业务线程池大小, 默认 200. 此外, 默认情况下线程池的队列大小是 Integer.MAX_VALUE.
* acceptCount: 实际上就是 tomcat backlog 大小, 默认 100, 可以参见 ServerSocketFactory.createSocket(int port, int backlog), 这个参数的作用不会体现在代码里

按照这个默认的参数的实际情况, 结合代码, 来分析一下情况. 假设我们的后端服务是一个无限挂起的服务, 请求过来以后如何处理

1. 请求数达到 200 以后, 线程池被占满, 同时达到 maxConnections 限制, 从代码上看, 达到 maxConnections 限制以后, Acceptor 进入 blocked 状态, 不再 accept socket
2. 请求继续增加, BacklogQueue 开始堆积, 这部分请求是无法得到处理的, 在连接建立后就被 hang 住
3. BacklogQueue 打到 100 以后, 开始出现 syn drop

按照这套配置, 业务线程池队列实际上是用不上的, 因为 maxConnections = maxThreads, 在一定程度上, BacklogQueue 充当了线程池队列的作用.

# 问题复现

为了复现问题, 我们做了一个小测试. 准备工作如下

1. 写一个接收请求后即无限挂起的 server
2. 调整 tomcat connector 参数如下

```xml
    <Connector port="9007" protocol="HTTP/1.1"
               connectionTimeout="20000"
               enableLookups="false" compression="on"
               redirectPort="8443"
               URIEncoding="UTF-8"
               maxThreads="10"
               maxConnections="15"
               acceptCount="10"
               compressableMimeType="text/html,text/xml,text/plain,text/javascript"
    />
```

开始测试, 使用 netstat 观察 syn drop 的情况
 
```bash
watch 'netstat -s | grep "sockets dropped"'
```

使用 ss 观察 receive queue 和 send queue

```bash
watch "ss -anlt | grep 9007"
```
ss 可以显示 9007 端口的 Recv-Q, 这个 queue 就是存储等待 accept 的 socket 的 BacklogQueue

初始状态下, sync drop 的情况和 Recv-Q 状态如下


```bash
Every 2.0s: ss -anlt | grep 9007                              Thu May 11 20:40:17 2017

LISTEN     0	  10           *:9007                     *:*
```

上面的 0 是 Recv-Q 目前的大小

```bash
Every 2.0s: netstat -s | grep "sockets dropped"               Thu May 11 20:39:55 2017

    297784 SYNs to LISTEN sockets dropped
```

使用一个异步 httpclient 每秒发送一个请求, 如下

```java
public class Test {
    @Test
    public void test() throws Exception {
        RateLimiter rateLimiter = RateLimiter.create(1);
        Builder builder = new Builder()
                .setConnectTimeout(1000)
                .setRequestTimeout(Integer.MAX_VALUE);
        AsyncHttpClient client = new AsyncHttpClient(builder.build());
        int count = 0;
        while (true) {
            rateLimiter.acquire();
            System.out.println("execute " + ++count);
            final int count0 = count;
            client.prepareGet("http://10.32.64.12:9007/message/block.htm").execute(
                    new AsyncCompletionHandler<Void>() {
                        @Override
                        public Void onCompleted(Response response) throws Exception {
                            System.out.println(
                                    "complete " + count0 + " " + response.getStatusCode() + ":"
                                            + response.getStatusText());
                            return null;
                        }

                        @Override
                        public void onThrowable(Throwable t) {
                            System.out.println(
                                    "error " + count0 + " " + t.getClass().getSimpleName());
                            super.onThrowable(t);
                        }
                    });
        }
    }
}
```

程序输出如下

```bash
execute 1
execute 2
execute 3
execute 4
execute 5
execute 6
execute 7
execute 8
execute 9
execute 10
execute 11
execute 12
execute 13
execute 14
execute 15
execute 16
execute 17
execute 18
execute 19
execute 20
execute 21
execute 22
execute 23
execute 24
execute 25
execute 26
execute 27
execute 28
execute 29
execute 30
execute 31
execute 32
execute 33
execute 34
execute 35
error 34 ConnectException
error 27 ConnectException
execute 36
execute 37
execute 38
error 37 ConnectException
error 26 ConnectException
execute 39
execute 40
error 32 ConnectException
error 28 ConnectException
execute 41
execute 42

... 省略后续输出
```

最终的 syn drop 的情况如下

```bash
Every 2.0s: ss -anlt | grep 9007                              Thu May 11 20:45:25 2017

LISTEN     11	  10           *:9007                     *:*
```

```bash
Every 2.0s: netstat -s | grep "sockets dropped"               Thu May 11 20:45:12 2017

    299578 SYNs to LISTEN sockets dropped
```

输出情况与预期符合

1. 请求到 10 的时候, 线程打满, 此时会开始入线程池队列
2. 请求到 15 的时候, maxConnections 打满, acceptor blocked
3. 请求到 25 的时候, Recv-Q 打满 (26 之前的请求都是挂起状态), 开始 syn drop, 连接开始失败, 日志开始报错 ConnectException (第 26 个开始)

# 总结

说到底, 还是服务端执行性能不够高, 才会导致这个问题啊, 还需要好好优化性能 :)