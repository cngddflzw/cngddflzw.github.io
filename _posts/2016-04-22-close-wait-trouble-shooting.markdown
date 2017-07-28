---
layout: post
title:  "CLOSE_WAIT 问题排查"
date:   2016-04-22
categories: tcp networking httpclient
---

* [问题背景](#h1)
* [问题分析](#h2)
* [解决问题](#h3)
* [测试代码](#h4)
* [题外话](#h5)

<a name="h1"></a>
# 问题背景
最近, 有好几个业务反馈试用 apache httpclient 时出现了问题, 使用的版本是 4.3.1, 表现出来的现象就是机器的连接监控出现大量的 close\_wait. 只有在 gc 的时候, 这些大量的连接才会被回收释放.

<a name="h2"></a>
# 问题分析

## tcp 四次挥手
要分析 close\_wait 形成的原因, 首先需要知道 tcp 连接在关闭时的流程, 如图:

![4 way handshake](https://upload.wikimedia.org/wikipedia/commons/5/55/TCP_CLOSE.svg)

tcp 连接的关闭可以由连接双方任何一方发起, 从图中可知, close\_wait 的状态发生在被动关闭方收到 fin, 并发出 ack 之后. 而在被动关闭方发出 fin 以后, 状态则会转为 last\_ack. close\_wait 的状态可能会永远持续下去, 直到连接被 close 状态变为 last\_ack.

由上我们已经知道, 造成这个现象的原因就是连接被对方关闭, 而自己没有发出 fin. 从 api 层面来讲, 就是没有调用 socket 的 close()

## 代码分析

查看业务代码, 发现业务的请求逻辑如下

``` java
public class RequestUtils {

    public static String get(String url) {
        CloseableHttpClient client = HttpClients.createDefault();
        HttpGet get = new HttpGet(url);
        CloseableHttpResponse response = null;
        try {
            response = client.execute(get);
            return EntityUtils.toString(response.getEntity());
        } catch (IOException e) {
            throw new RuntimeException(e);
        } finally {
            if (response != null) {
                try {
                    response.close();
                } catch (IOException e1) {
                    // ignored
                }
            }
        }
    }
}
```

看起来似乎没有什么问题, 最后已经调用了 response.close(), 看起来连接应该已经被关闭了, 但真的是这样吗? 既然连接是处于 close\_wait 状态, 那么说明代码并有调用 socket.close(). 跟踪 response.close() 的代码, 到底层后会发现, close 时有一个判断逻辑, 即 Response 的 header 中有 Connection: keep-alive, 则连接不会被关闭, 以便下次请求同一个网站的时候复用连接. 所以, 问题就出在这.

## 问题原因
连接没有主动关闭, 是造成 close\_wait 的直接原因. 那么更本原因是什么? 以前我们总是会说, httpclient 要复用, 连接要复用, 到底是为什么, 不复用会造成什么潜在的问题, 其实就是这个原因. 以上业务代码的问题就在于每次调用这个方法都会创建一个新的 HttpClient, 他的问题在于:
    1. 浪费资源, 每次需要重新初始化 client
    2. 由于 keep-alive 的存在 (http/1.1 协议默认开启 keep-alive), 导致连接不会被关闭, 只会被回收
    3. 局部方法内生成 client, 在 finally 内没有关闭 client, 导致连接的泄露
实际上, 上面出现 close\_wait 的应该是在 server 那边 hang 住连接超过一定时间以后超时, server 才关闭连接的, 因为既然 server 返回 keep-alive, 说明 server 实际上也期待客户端能复用连接, 却没想到客户端把连接给泄露了.

<a name="h3"></a>
# 解决问题

知道了问题原因, 如何改进业务代码? 改进之前, 应该清楚使用 httpclient 的一些原则:
    1. 复用 HttpClient 避免重复初始化
    2. 使用连接池管理连接, 避免每次新建连接
    3. 设置合理的连接池大小, 以及连接存活时间
改进后的代码如下:

``` java
public class RequestUtils2 {

    private static final CloseableHttpClient CLIENT;

    static {
        PoolingHttpClientConnectionManager connectionManager = 
        		new PoolingHttpClientConnectionManager(60, TimeUnit.SECONDS);
        connectionManager.setMaxTotal(1080);
        connectionManager.setDefaultMaxPerRoute(128);

        CLIENT = HttpClientBuilder.create()
                .setConnectionManager(connectionManager)
                .build();
    }

    public static String get(String url) {
        HttpGet get = new HttpGet(url);
        CloseableHttpResponse response = null;
        try {
            response = CLIENT.execute(get);
            return EntityUtils.toString(response.getEntity());
        } catch (IOException e) {
            throw new RuntimeException(e);
        } finally {
            if (response != null) {
                try {
                    response.close();
                } catch (IOException e1) {
                    // ignored
                }
            }
        }
    }
}
```

<a name="h4"></a>
# 测试代码 
有想测试这个问题的同学, 可以用以下代码启动一个 httpserver
``` java
public class SimpleHttpServer {

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8000), 0);
        server.createContext("/test", new MyHandler());
        server.setExecutor(null); // creates a default executor
        server.start();
    }

    static class MyHandler implements HttpHandler {

        @Override
        public void handle(HttpExchange t) throws IOException {
            String response = "This is the response";
            Headers headers = t.getResponseHeaders();
            headers.add("Connection", "keep-alive");
            t.sendResponseHeaders(200, response.length());
            OutputStream os = t.getResponseBody();
            os.write(response.getBytes());
            os.close();
        }
    }
}
```
然后使用错误代码请求
``` java
public class RequestUtilsTest {

    @Test
    public void testRequest() throws Exception {
        String s = RequestUtils.get("http://localhost:8000/test");
        System.out.println(s);
        Thread.sleep(Long.MAX_VALUE);
    }
}
```
等到请求结束后, 手动结束 httpserver, 再用 sudo netstat -antup (Mac 使用 lsof -i -P) 查看一下端口状态, 你会发现如下信息
```
java      45896     liuzhenwei   48u  IPv6 0x8c89b842b9402bf3      0t0    TCP localhost:52519->localhost:8000 (CLOSE_WAIT)
```

<a name="h5"></a>
# 题外话 
有人会疑问, server 端关闭连接后, client 需要手动关闭连接吗? 关于这一点, 可以参考写简单的 socket 业务逻辑时的代码, 如果被动关闭方需要发出 fin, 是需要调用 socket.close() 的. 例如, 一般我们会这么写:

``` java
        Socket socket = new Socket("localhost", 9000);
        InputStream in = socket.getInputStream();
        while (true) {
            int read = in.read();
            if (read == -1) { // 这个 -1 就是 server close 连接后发出的
                in.close();
                break;
            }
            System.out.println(read);
        }
```

也就是说, 主动关闭方如果 close 了, 那么被动关闭方这边状态会变为 close\_wait, 同时收到 -1, 然后我们需要手动调用 socket.close().

