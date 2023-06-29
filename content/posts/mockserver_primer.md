---
title: okhttp自带的mockserver教程
date: 2019-10-13 21:05:57
tags: [mock]
categories: [HTTP]
---

## 概述
本篇记录okhttp自带的mockserver这个库的使用方式.

作为一个网络库,okhttp自身也实现了一个mockserver,以方便写测试用例,这个库是独立的,也可以单独使用.

用作平时简单的mock数据,进行测试,很方便

## 使用方式 
此处以Android为例,Java除了依赖方式有点差异,其他一致;

1. 添加依赖

```groovy
    androidTestImplementation('com.squareup.okhttp3:mockwebserver:3.13.1')
```

2. 代码使用

```java
    public class MockRes {

    public final MockWebServer server = new MockWebServer();
    public OkHttpClient client = new OkHttpClient();

    @Test
    public void simpleTest() {
        // 构造一个mock的 response
        MockResponse mockResponse = new MockResponse().setBody("abc");
        // 添加到 server 中,server中将会按照FIFO的方式进行返回
        // enqueue 一个,下次请求就返回队列中最靠前的,是同步的
        server.enqueue(mockResponse);

        // 同步发起请求,虽然此处添加了path,实则在不自定义dispatcher的请求下,是不会影响前一步mockresponse的返回的
        Response executeRes = executeSynchronously("/a");

        // 消费mock的结果
        assertNotNull(executeRes);
        try {
            assertEquals("abc", executeRes.body().string());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    @Test
    public void customDispatch() {
        // 采用自定义 dispatcher 之后,就不能再 调用 server.enqueue(mockResponse) 方法,所有的mock行为均定义在 Dispatcher 类中
        server.setDispatcher(new Dispatcher() {
            @Override
            public MockResponse dispatch(RecordedRequest request) throws InterruptedException {
                if ("/a".equals(request.getPath())) {
                    return new MockResponse().setBody("A");
                }
                return new MockResponse().setBody("O");
            }
        });

        // 同步发起请求,path 为 /a 怎么应该返回body 是 A
        Response executeRes = executeSynchronously("/a");

        // 消费mock的结果
        assertNotNull(executeRes);
        try {
            assertEquals("A", executeRes.body().string());
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 同步发起请求,path 为 /b 怎么应该返回body 是 O
        Response bRes = executeSynchronously("/b");

        // 消费mock的结果
        assertNotNull(bRes);
        try {
            assertEquals("O", bRes.body().string());
        } catch (IOException e) {
            e.printStackTrace();
        }

    }


    private Response executeSynchronously(String path, String... headers) {
        Request.Builder builder = new Request.Builder();
        builder.url(server.url(path));
        for (int i = 0, size = headers.length; i < size; i += 2) {
            builder.addHeader(headers[i], headers[i + 1]);
        }
        Call call = client.newCall(builder.build());
        try {
            return call.execute();
        } catch (IOException e) {
            return null;
        }
    }

}

```



## 参考
1. [okhttp测试](https://github.com/square/okhttp/blob/okhttp_3.13.x/okhttp-tests/src/test/java/okhttp3/CallTest.java)