---
layout: post
title:  "从 tcp drop 看 tomcat 线程模型"
date:   2017-05-11
categories: network tcp tomcat
---

# 问题背景

近期在业务高峰期, 发现会出现客户端请求发送到服务端无法响应的情况. 通过查看机器监控, 我们发现出现了很多 syn drop 的情况.

# SYN Drop

要想查找问题根源所在, 首先需要理解 syn drop 的含义和成因. 

在 tcp 的连接过程中, 会用到一个叫 *BacklogQueue* 的队列, 这个队列用于存储那些三次握手成功后, 处于*等待accept*状态的 socket, accept queue 的大小由操作系统参数 backlog 决定. 每次 server 端应用程序调用 socket.accept(), 就会从 BacklogQueue 获取一个 socket. 一旦这个队列满了以后, 就会出现 syn drop 的情况, 即三次握手的 syn 包被直接丢弃, 不给客户端任何回复, 这样就会导致客户端出现连接无法建立的情况. 

# 应用的问题

不难发现, 出现 syn drop 的原因是因为应用没有及时的 accept socket, 导致 BacklogQueue 的堆积. 对应到我们的服务端应用, 实际上就是 tomcat 对连接的处理程序. 由于我们的 tomcat 基本都是使用的 bio acceptor, 下面看 bio 的线程模型.

![](/_posts/.2017-05-11-sync-drop-and-tomcat-thread-model.markdown_images/89af2252.png)

tomcat 有一个专门的 Acceptor 的用来 accept socket, 它是一个不停的死循环线程, 不断的 accept socket, 并将 socket 派发给线程池去处理实际的数据. 结合实际的 acceptor 代码实现

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
                    // 参数 maxConnections
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
                        // 参数 maxThreads
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

从代码上看, 这个模型相对来说还是比较简单明了, 但这里会涉及到几个关键参数

* maxConnections: tomcat 最大并发连接数, 默认 200
* maxThreads: tomcat 业务线程池大小, 默认 200. 此外, 默认情况下线程池的队列大小是 Integer.MAX_VALUE.
* acceptCount: 实际上就是 tomcat backlog 大小, 默认 100, 可以参见 ServerSocketFactory.createSocket(int port, int backlog), 这个参数的作用不会体现在代码里

按照这个默认的参数的实际情况, 结合代码, 来分析一下情况. 假设我们的后端服务是一个无限挂起的服务, 请求过来以后如何处理

1. 请求数达到 200 以后, 线程池被占满, 同时达到 maxConnections 限制, 从代码上看, 达到 maxConnections 限制以后, Acceptor 进入 blocked 状态, 不再 accept socket
2. 请求继续增加, BacklogQueue 开始堆积, 这部分请求是无法得到处理的, 在连接建立后就被 hang 住
3. BacklogQueue 打到 100 以后, 开始出现 syn drop

按照这套配置, 业务线程池队列实际上是用不上的, 因为 maxConnections = maxThreads, 在一定程度上, BacklogQueue 充当了线程池队列的作用

----

参考