---
title: jdk携带的一个HttpServer实现
tags:
  - Android
  - Java
categories:
  - Android
date: 2020-04-20 18:57:56
---
## 概述
记录一个意外发现的一个类 `com.sun.net.httpserver.HttpsServer`. 一个 Http 的 Server 端.

<!-- more -->

## 用处
1. 适用于泛前端类开发者,在无后端服务的情况下,可以用来mock数据或者mock后端行为,非常灵活.
2. 适用于网络库的开发者,测试库的行为;

## 缺点
1. 目前不支持`HTTP2`协议.

## 分类
### HTTP 协议

#### 自定义一个 HTTP 服务;

```java
        HttpsServer server = HttpsServer.create(new InetSocketAddress(8500), 0);
        HttpsConfigurator httpsConfigurator = new HttpsConfigurator(SSLContext.getDefault());
        server.setHttpsConfigurator(httpsConfigurator);
        HttpContext context = server.createContext("/example");
        context.setHandler(new CustomHttpHandler());
        server.start();
```

该 Http 服务,是在本机的 `8500` 端口启动的; 根目录为 `example`. 所以,直接通过 `http://127.0.0.1:8500/example` 即可访问.

#### Server 的行为定义

```java
    public class CustomHttpHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            URI requestURI = exchange.getRequestURI();
            printRequestInfo(exchange);
            String response = "This is the response at " + requestURI;

            exchange.sendResponseHeaders(200, 0);

            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        }

        private void printRequestInfo(HttpExchange exchange) {
            System.out.println("-- headers --");
            Headers requestHeaders = exchange.getRequestHeaders();
            requestHeaders.entrySet().forEach(System.out::println);

            System.out.println("-- principle --");
            HttpPrincipal principal = exchange.getPrincipal();
            System.out.println(principal);

            System.out.println("-- HTTP method --");
            String requestMethod = exchange.getRequestMethod();
            System.out.println(requestMethod);

            System.out.println("-- query --");
            URI requestURI = exchange.getRequestURI();
            String query = requestURI.getQuery();
            System.out.println(query);


            InputStream requestBody = exchange.getRequestBody();
            if (requestBody == null) {
                return;
            }
            int available = 0;
            try {
                available = requestBody.available();
            } catch (IOException e) {
                e.printStackTrace();
            }
            System.out.println("request body available:" + available);
            printMessage(requestBody);
        }
        private static void printMessage(InputStream requestBody) {
            byte[] buffer = new byte[1024];
            try {
                while (true) {
                    int read = requestBody.read(buffer);
                    if (!(read > 0)) break;
                    System.out.println("body:::::" + new String(buffer));
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

### Https

> 生成自签的证书的命令如下: 
> keytool -genkey -alias alias -keyalg RSA -keystore keystore.jks -keysize 2048


#### 自定义一个 Https 服务;

```java
        try {
            // setup the socket address
            InetSocketAddress address = new InetSocketAddress(8500);

            // initialise the HTTPS server
            HttpsServer httpsServer = HttpsServer.create(address, 0);
            SSLContext sslContext = SSLContext.getInstance("TLS");

            // initialise the keystore
            // 记得替换密码
            char[] password = "123456".toCharArray();
            KeyStore ks = KeyStore.getInstance("JKS");
            FileInputStream fis = new FileInputStream("keystore.jks");
            ks.load(fis, password);

            // setup the key manager factory
            KeyManagerFactory kmf = KeyManagerFactory.getInstance("SunX509");
            kmf.init(ks, password);

            // setup the trust manager factory
            TrustManagerFactory tmf = TrustManagerFactory.getInstance("SunX509");
            tmf.init(ks);

            // setup the HTTPS context and parameters
            sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);
            httpsServer.setHttpsConfigurator(new HttpsConfigurator(sslContext) {
                public void configure(HttpsParameters params) {
                    try {
                        // initialise the SSL context
                        SSLContext context = getSSLContext();
                        SSLEngine engine = context.createSSLEngine();
                        params.setNeedClientAuth(false);
                        params.setCipherSuites(engine.getEnabledCipherSuites());
                        params.setProtocols(engine.getEnabledProtocols());

                        // Set the SSL parameters
                        SSLParameters sslParameters = context.getSupportedSSLParameters();
                        params.setSSLParameters(sslParameters);

                    } catch (Exception ex) {
                        System.out.println("Failed to create HTTPS port");
                    }
                }
            });
            httpsServer.createContext("/example", new SimpleHttpsServer.SimpleHandler());
            httpsServer.start();

        } catch (Exception exception) {
            System.out.println("Failed to create HTTPS server on port " + 8500 + " of localhost");
            exception.printStackTrace();
        }
```


#### 自定义Https服务处理

```java
      public class SimpleHttpsServer {

          public static class SimpleHandler implements HttpHandler {
              @Override
              public void handle(HttpExchange t) throws IOException {
                  printRequestInfo(t);
                  String response = "This is the response";
                  HttpsExchange httpsExchange = (HttpsExchange) t;
                  httpsExchange.getResponseHeaders().add("Access-Control-Allow-Origin", "*");
                  httpsExchange.sendResponseHeaders(200, response.getBytes().length);
                  OutputStream os = httpsExchange.getResponseBody();
                  os.write(response.getBytes());
                  os.close();
              }
          }

          private static void printRequestInfo(HttpExchange exchange) {
              System.out.println("-- headers --");
              Headers requestHeaders = exchange.getRequestHeaders();
              requestHeaders.entrySet().forEach(System.out::println);

              System.out.println("-- protocol --");
              String protocol = exchange.getProtocol();
              System.out.println(protocol);


              System.out.println("-- principle --");
              HttpPrincipal principal = exchange.getPrincipal();
              System.out.println(principal);

              System.out.println("-- HTTP method --");
              String requestMethod = exchange.getRequestMethod();
              System.out.println(requestMethod);

              System.out.println("-- query --");
              URI requestURI = exchange.getRequestURI();
              String query = requestURI.getQuery();
              System.out.println(query);


              InputStream requestBody = exchange.getRequestBody();
              if (requestBody == null) {
                  return;
              }
              int available = 0;
              try {
                  available = requestBody.available();
              } catch (IOException e) {
                  e.printStackTrace();
              }
              System.out.println("request body available:" + available);
              printMessage(requestBody);
          }

          private static void printMessage(InputStream requestBody) {
              byte[] buffer = new byte[1024];
              try {
                  while (true) {
                      int read = requestBody.read(buffer);
                      if (!(read > 0)) break;
                      System.out.println("body content is: " + new String(buffer));
                  }
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
      }
```

## 末尾

如果知道实现了Http2的,还望告知.