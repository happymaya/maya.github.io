---
title: java 以及 tocmat 使用正向代理案例
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-11-02 11:34:00 +0800
categories:  [Blog, Notes]
tags: [proxy, 正向代理, 反向代理, java, tomcat]
---

最近部署的应用服务器无法直接访问互联网, 只好在前置服务器上配置了 squid 做正向代理, 应用采用java语言, 这里回顾总结下 java 以及 tomcat 使用http正向代理的几种方法：

squid使用3128端口

# java 正向代理设置
## 设置 java 的启动参数

```bash
# http 代理 
-Dhttp.proxyHost=proxy server ip -Dhttp.proxyPort=proxy server ip port
# https 代理
-Dhttps.proxyHost=proxy server ip -Dhttps.proxyPort=proxy server ip port
# https代理

比如，在 tomcat 的 /bin/catalic.sh 下加入下面的代码：
```bash

```

## 在 java 代码初始化时设置环境变量
- http 代理
```bash
System.setProperty("http.proxyHost", "代理ip");
System.setProperty("http.proxyPort", "代理ip端口`");

- https 代理
```bash
System.setProperty("https.proxyHost", "代理ip");
System.setProperty("https.proxyPort", "3128");
```

 
## 在 java 代码中设置使用代理：
```bash
URL url = new URL("https://某网址");
Proxy proxy = new Proxy(Proxy.Type.DIRECT.HTTP, new InetSocketAddress("代理ip", 3128));  
HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);
```

## 如果操作系统已经配置好代理，可以直接使用
```bash
System.setProperty("java.net.useSystemProxies", "true");
; 当然也可以在启动时增加 -Djava.net.useSystemProxies=true

; 如果某些网址不需要使用代理，可以单独进行设置，比如：
-Dhttp.nonProxyHosts="www.hongxuejing.com|localhost"
```

# httpPost设置代理
```java
public JSONObject httpPost(String url, JSONObject jsonParam) {
        // post请求返回结果
        CloseableHttpClient httpClient = HttpClients.createDefault();
        JSONObject jsonResult = null;
        HttpPost httpPost = new HttpPost(url);
        HttpHost target = new HttpHost("qa.chetong.net/ailoss/getAiLossPageURL", 8080,  
                "http");
        HttpHost proxy = new HttpHost("10.1.200.95", 3128, "http");
        // 设置请求和传输超时时间
        RequestConfig requestConfig = RequestConfig.custom().setSocketTimeout(2000).setConnectTimeout(2000).setProxy(proxy).build();
        httpPost.setConfig(requestConfig);
        try {
            if (null != jsonParam) {
                // 解决中文乱码问题
                StringEntity entity = new StringEntity(jsonParam.toString(), "utf-8");
                entity.setContentEncoding("UTF-8");
                entity.setContentType("application/json");
                httpPost.setEntity(entity);
            }
            CloseableHttpResponse result = httpClient.execute(target,httpPost);
            // 请求发送成功，并得到响应
            if (result.getStatusLine().getStatusCode() == HttpStatus.SC_OK) {
                String str = "";
                try {
                    // 读取服务器返回过来的json字符串数据
                    str = EntityUtils.toString(result.getEntity(), "utf-8");
                    // 把json字符串转换成json对象
                    jsonResult = JSONObject.parseObject(str);
                } catch (Exception e) {
                    log.error("post请求提交失败:" + url, e);
                }
            }
        } catch (IOException e) {
            log.error("post请求提交失败:" + url, e);
            System.out.println(e);
        } finally {
            httpPost.releaseConnection();
        }
        return jsonResult;
    }

//测试main方法
public static void main(String[] args) {
        String url = "";

        String json = "";
        HttpClientUtil httpClient = new HttpClientUtil();
        JSONObject jsonObject = httpClient.httpPost(url,json);

}
```

# curl 正向代理
常用切支持 http(s) 协议代理主要分为两大类: http 代理和 socks 代理，见下表：
| 大类       | 小类               | 子类                                 | 描述                                                         |
| ---------- | ------------------ | ------------------------------------ | ------------------------------------------------------------ |
| http 代理  | http代理<br/> https代理 | 透明代理                             | http服务器知道浏览器端使用了代理，并能获取浏览器端原始IP；   |
|            |                    | 匿名代理                             | http服务器知道浏览器端使用了代理，但无法获取浏览器端原始IP； |
|            |                    | 高匿名代理                           | http服务器不知道浏览器端使用了代理，且无法获取浏览器端原始IP； |
| SOCKS 代理 | SOCKS4             | 被称为全能代 理，支持http 和其他协议 | 只支持TCP应用；                                              |
|            | SOCKS4A            |                                      | 支持TCP应用；支持服务器端域名解析；                          |
|            | SOCKS5             |                                      | 支持TCP和UDP应用；支持服务器端域名解析； 支持多种身份验证；支持IPV6； |

## Linux curl 命令设置代理举例
linux curl命令可以使用下面参数设置http(s)代理、socks代理, 已经设置它们的用户名、密码以及认证方式：
| 参数                                                         | 用法                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| -x host:port<br/>-x [protocol://[user:pwd@]host[:port]<br/>-proxy [protocol://[user:pwd@]host[:port] | -x host:port<br/>-x [protocol://[user:pwd@]host[:port]<br/>–proxy [protocol://[user:pwd@]host[:port] |
| –socks4 <host[:port]><br/>–socks4a <host[:port]><br/>–socks5 <host[:port]> | 使用SOCKS4代理；<br/>使用SOCKS4A代理；<br/>使用SOCKS5代理；<br/>此参数会覆盖“-x”参数； |
| –proxy-anyauth<br/>–proxy-basic<br/>–proxy-diges<br/>–proxy-negotiate<br/>–proxy-ntlm | –proxy-anyauth<br/>–proxy-basic<br/>–proxy-diges<br/>–proxy-negotiate<br/>–proxy-ntlm |
| -U user:password<br/>–proxy-user user:password               | 设置代理的用户名和密码；                                     |

### Linux curl 命令设置 http 代理
```bash
# 指定http代理IP和端口
curl -x 113.185.19.192:80 http://aiezu.com/test.php
curl --proxy 113.185.19.192:80 http://aiezu.com/test.php
 
#指定为http代理
curl -x http_proxy://113.185.19.192:80 http://aiezu.com/test.php
 
#指定为https代理
curl -x HTTPS_PROXY://113.185.19.192:80 http://aiezu.com/test.php
 
#指定代理用户名和密码，basic认证方式
curl -x aiezu:123456@113.185.19.192:80 http://aiezu.com/test.php
curl -x 113.185.19.192:80 -U aiezu:123456 http://aiezu.com/test.php
curl -x 113.185.19.192:80 --proxy-user aiezu:123456 http://aiezu.com/test.php
 
#指定代理用户名和密码，ntlm认证方式
curl -x 113.185.19.192:80 -U aiezu:123456 --proxy-ntlm http://aiezu.com/test.php
 
#指定代理协议、用户名和密码，basic认证方式
curl -x http_proxy://aiezu:123456@113.185.19.192:80 http://aiezu.com/test.php

```

### Linux curl 命令设置 socks 代理
```bash
#使用socks4代理，无需认证方式
curl --socks4 122.192.32.76:7280 http://aiezu.com/test.php
curl -x socks4://122.192.32.76:7280 http://aiezu.com/test.php
 
#使用socks4a代理，无需认证方式
curl --socks4a 122.192.32.76:7280 http://aiezu.com/test.php
curl -x socks4a://122.192.32.76:7280 http://aiezu.com/test.php
 
#使用socks5代理，basic认证方式
curl --socks5 122.192.32.76:7280 -U aiezu:123456 http://aiezu.com/test.php
curl -x socks5://aiezu:123456@122.192.32.76:7280 http://aiezu.com/test.php
 
#使用socks5代理，basic认证方式，ntlm认证方式
curl -x socks5://aiezu:123456@122.192.32.76:7280 --proxy-ntlm http://aiezu.com/test.php

```

### curl 设置代理 post 方式
```bash
curl -H "Content-Type: application/json" -X POST -x http_proxy://代理ip:端口 请求地址
```
