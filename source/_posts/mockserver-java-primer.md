---
title: mockserver_java_primer
date: 2019-10-14 13:51:08
tags: [mock]
categories: [HTTP]
---
# 概述
记录另外一个mockserver的库使用方式. API 更加丰富.
<!-- more -->

# 添加依赖

```groovy
compile group: 'org.mock-server', name: 'mockserver-netty', version: '5.6.1'
compile group: 'log4j', name: 'log4j', version: '1.2.17'
```

# 使用方式
## 最简单的使用方式, 请求 -> 返回mock的response

- Server端

```java
public class MockServerTest {

    public static void main(String[] args) {
        // 1. 8000端口启动服务
        ClientAndServer.startClientAndServer(8000);

        // 2. new 一个操作服务端行为的实例
        MockServerClient serverClient = new MockServerClient("localhost", 8000);

        // 3. 定义服务端的行为
        serverClient
                .when(request()
                        .withMethod("GET")
                        .withPath("/path1/function1"))
                .respond(response()
                        .withStatusCode(200)
                        .withBody("body200"));


        serverClient
                .when(request()
                        .withMethod("GET")
                        .withPath("/path2/function2")
                        .withCookies(cookie("session", "4930456C-C718-476F-971F-CB8E047AB349"))
                        .withQueryStringParameters(param("cartId", "055CA455-1DF7-45BB-8535-4F83E7266092")))
                .respond(response()
                        .withStatusCode(307)
                        .withBody("body307"));

    }

}
```

- Client 端

```java
public class MockTestClient {

    public static void main(String[] args) {

        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder().url("http://localhost:8000/path1/function1")
                .build();
        Response response = null;

        try {
            response = client.newCall(request).execute();
            System.out.println(response.body().string());
        } catch (
                IOException e) {
            e.printStackTrace();
        }
    }

}

```

以上先记录最简单的使用,还有forward,callback,verify,retrieve,感觉用的不多,暂不记录,需要的时候再说吧.


# 参考
1. [mockserverRepo](https://github.com/jamesdbloom/mockserver)
2. [mockserverPage](http://www.mock-server.com/#what-is-mockserver)