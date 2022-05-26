---
title: RestTemplate 远程调用实现原理
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-05-07 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, RestTemplate]
---

RestTemplate 访问 HTTP 端点的使用方法，它涉及 RestTemplate 初始化、发起请求及获取响应结果等核心环节。


## 初始化 RestTemplate 实例

通过 RestTemplate 提供的几个构造函数对 RestTemplate 进行初始化。在分析这些构造函数之前，有必要先看一下 RestTemplate 类的定义，如下代码所示：
```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperation{}
```

从上述代码中，可以看到 RestTemplate 扩展了 InterceptingHttpAccessor 抽象类，并实现了 RestOperations 接口。接下来围绕 RestTemplate 的方法定义进行设计思路的梳理。

首先，来看看 RestOperations 接口的定义，这里截取了部分核心方法，如下代码所示：
```java
public interface RestOperations {
    @Nullable
    <T> T getForObject(String var1, Class<T> var2, Object... var3) throws RestClientException;

    @Nullable
    <T> T getForObject(String var1, Class<T> var2, Map<String, ?> var3) throws RestClientException;

    @Nullable
    <T> T getForObject(URI var1, Class<T> var2) throws RestClientException;

    <T> ResponseEntity<T> getForEntity(String var1, Class<T> var2, Object... var3) throws RestClientException;

    <T> ResponseEntity<T> getForEntity(String var1, Class<T> var2, Map<String, ?> var3) throws RestClientException;

    <T> ResponseEntity<T> getForEntity(URI var1, Class<T> var2) throws RestClientException;

    HttpHeaders headForHeaders(String var1, Object... var2) throws RestClientException;

    HttpHeaders headForHeaders(String var1, Map<String, ?> var2) throws RestClientException;

    HttpHeaders headForHeaders(URI var1) throws RestClientException;

    @Nullable
    URI postForLocation(String var1, @Nullable Object var2, Object... var3) throws RestClientException;

    @Nullable
    URI postForLocation(String var1, @Nullable Object var2, Map<String, ?> var3) throws RestClientException;

    @Nullable
    URI postForLocation(URI var1, @Nullable Object var2) throws RestClientException;

    @Nullable
    <T> T postForObject(String var1, @Nullable Object var2, Class<T> var3, Object... var4) throws RestClientException;

    @Nullable
    <T> T postForObject(String var1, @Nullable Object var2, Class<T> var3, Map<String, ?> var4) throws RestClientException;

    @Nullable
    <T> T postForObject(URI var1, @Nullable Object var2, Class<T> var3) throws RestClientException;

    <T> ResponseEntity<T> postForEntity(String var1, @Nullable Object var2, Class<T> var3, Object... var4) throws RestClientException;

    <T> ResponseEntity<T> postForEntity(String var1, @Nullable Object var2, Class<T> var3, Map<String, ?> var4) throws RestClientException;

    <T> ResponseEntity<T> postForEntity(URI var1, @Nullable Object var2, Class<T> var3) throws RestClientException;

    void put(String var1, @Nullable Object var2, Object... var3) throws RestClientException;

    void put(String var1, @Nullable Object var2, Map<String, ?> var3) throws RestClientException;

    void put(URI var1, @Nullable Object var2) throws RestClientException;

    @Nullable
    <T> T patchForObject(String var1, @Nullable Object var2, Class<T> var3, Object... var4) throws RestClientException;

    @Nullable
    <T> T patchForObject(String var1, @Nullable Object var2, Class<T> var3, Map<String, ?> var4) throws RestClientException;

    @Nullable
    <T> T patchForObject(URI var1, @Nullable Object var2, Class<T> var3) throws RestClientException;

    void delete(String var1, Object... var2) throws RestClientException;

    void delete(String var1, Map<String, ?> var2) throws RestClientException;

    void delete(URI var1) throws RestClientException;

    Set<HttpMethod> optionsForAllow(String var1, Object... var2) throws RestClientException;

    Set<HttpMethod> optionsForAllow(String var1, Map<String, ?> var2) throws RestClientException;

    Set<HttpMethod> optionsForAllow(URI var1) throws RestClientException;

    <T> ResponseEntity<T> exchange(String var1, HttpMethod var2, @Nullable HttpEntity<?> var3, Class<T> var4, Object... var5) throws RestClientException;

    <T> ResponseEntity<T> exchange(String var1, HttpMethod var2, @Nullable HttpEntity<?> var3, Class<T> var4, Map<String, ?> var5) throws RestClientException;

    <T> ResponseEntity<T> exchange(URI var1, HttpMethod var2, @Nullable HttpEntity<?> var3, Class<T> var4) throws RestClientException;

    <T> ResponseEntity<T> exchange(String var1, HttpMethod var2, @Nullable HttpEntity<?> var3, ParameterizedTypeReference<T> var4, Object... var5) throws RestClientException;

    <T> ResponseEntity<T> exchange(String var1, HttpMethod var2, @Nullable HttpEntity<?> var3, ParameterizedTypeReference<T> var4, Map<String, ?> var5) throws RestClientException;

    <T> ResponseEntity<T> exchange(URI var1, HttpMethod var2, @Nullable HttpEntity<?> var3, ParameterizedTypeReference<T> var4) throws RestClientException;

    <T> ResponseEntity<T> exchange(RequestEntity<?> var1, Class<T> var2) throws RestClientException;

    <T> ResponseEntity<T> exchange(RequestEntity<?> var1, ParameterizedTypeReference<T> var2) throws RestClientException;

    @Nullable
    <T> T execute(String var1, HttpMethod var2, @Nullable RequestCallback var3, @Nullable ResponseExtractor<T> var4, Object... var5) throws RestClientException;

    @Nullable
    <T> T execute(String var1, HttpMethod var2, @Nullable RequestCallback var3, @Nullable ResponseExtractor<T> var4, Map<String, ?> var5) throws RestClientException;

    @Nullable
    <T> T execute(URI var1, HttpMethod var2, @Nullable RequestCallback var3, @Nullable ResponseExtractor<T> var4) throws RestClientException;
}
```

显然，RestOperations 接口定义了 get/post/put/delete/exhange 等所有远程调用方法组，这些方法都遵循 RESTful 架构风格而设计。RestTemplate 为这些接口提供了实现机制，这是它的一条代码支线。

InterceptingHttpAccessor，是一个抽象类，包含的核心变量如下代码所示：
```java
public abstract class InterceptingHttpAccessor extends HttpAccessor {
    private final List<ClientHttpRequestInterceptor> interceptors = new ArrayList();
    @Nullable
    private volatile ClientHttpRequestFactory interceptingRequestFactory;

    public InterceptingHttpAccessor() {
    }

    public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
        if (this.interceptors != interceptors) {
            this.interceptors.clear();
            this.interceptors.addAll(interceptors);
            AnnotationAwareOrderComparator.sort(this.interceptors);
        }

    }

    public List<ClientHttpRequestInterceptor> getInterceptors() {
        return this.interceptors;
    }

    public void setRequestFactory(ClientHttpRequestFactory requestFactory) {
        super.setRequestFactory(requestFactory);
        this.interceptingRequestFactory = null;
    }

    public ClientHttpRequestFactory getRequestFactory() {
        List<ClientHttpRequestInterceptor> interceptors = this.getInterceptors();
        if (!CollectionUtils.isEmpty(interceptors)) {
            ClientHttpRequestFactory factory = this.interceptingRequestFactory;
            if (factory == null) {
                factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
                this.interceptingRequestFactory = (ClientHttpRequestFactory)factory;
            }

            return (ClientHttpRequestFactory)factory;
        } else {
            return super.getRequestFactory();
        }
    }
}
```

通过变量定义，明确了 InterceptingHttpAccessor 包含两部分处理功能：
- 一部分负责设置和管理请求拦截器 ClientHttpRequestInterceptor；
- 另一部分负责获取用于创建客户端 HTTP 请求的工厂类 ClientHttpRequestFactory。

同时，注意到 InterceptingHttpAccessor 同样存在一个父类 HttpAccessor，这个父类值真正实现了 ClientHttpRequestFactory 的创建及如何通过 ClientHttpRequestFactory 获取代表客户端请求的 ClientHttpRequest 对象。HttpAccessor 的核心变量如下代码所示：
```java
public abstract class HttpAccessor {
    protected final Log logger = HttpLogging.forLogName(this.getClass());
    private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
    private final List<ClientHttpRequestInitializer> clientHttpRequestInitializers = new ArrayList();

    public HttpAccessor() {
    }

    public void setRequestFactory(ClientHttpRequestFactory requestFactory) {
        Assert.notNull(requestFactory, "ClientHttpRequestFactory must not be null");
        this.requestFactory = requestFactory;
    }

    public ClientHttpRequestFactory getRequestFactory() {
        return this.requestFactory;
    }

    public void setClientHttpRequestInitializers(List<ClientHttpRequestInitializer> clientHttpRequestInitializers) {
        if (this.clientHttpRequestInitializers != clientHttpRequestInitializers) {
            this.clientHttpRequestInitializers.clear();
            this.clientHttpRequestInitializers.addAll(clientHttpRequestInitializers);
            AnnotationAwareOrderComparator.sort(this.clientHttpRequestInitializers);
        }

    }

    public List<ClientHttpRequestInitializer> getClientHttpRequestInitializers() {
        return this.clientHttpRequestInitializers;
    }

    protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
        ClientHttpRequest request = this.getRequestFactory().createRequest(url, method);
        this.initialize(request);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("HTTP " + method.name() + " " + url);
        }

        return request;
    }

    private void initialize(ClientHttpRequest request) {
        this.clientHttpRequestInitializers.forEach((initializer) -> {
            initializer.initialize(request);
        });
    }
}
```
从以上代码可以看到，HttpAccessor 中创建了 SimpleClientHttpRequestFactory 作为系统默认的 ClientHttpRequestFactory。

由此可见，RestTemplate 的类层结构，如下图所示：
![RestTemplate 的类层次结构](http://processon.com/chart_image/62865210f346fb4244cc02b3.png)

在 RestTemplate 的类层结构中，就能快速理解它的设计思想。整个类层结构清晰地分成两条支线：
- 左边支线用于完成与 HTTP 请求相关的实现机制；
- 而右边支线提供了基于 RESTful 风格的操作入口，并使用了面向对象中的接口和抽象类完成这两部分功能的聚合。

## RestTemplate 核心执行流程

作为用于远程调用的模板工具类，可以从具备多种请求方式的 exchange 方法入手，该方法的定义如下代码所示：
```java
public <T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, Class<T> responseType, Object... uriVariables) throws RestClientException {
    // 构建请求回调
    RequestCallback requestCallback = this.httpEntityCallback(requestEntity, responseType);
    // 构建响应体抽取器
    ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
    // 执行远程调用
    return (ResponseEntity)nonNull(this.execute(url, method, requestCallback, responseExtractor, uriVariables));
}
```

显然，应该进一步关注这里的 execute 方法。事实上，无论采用 get/put/post/delete 中的哪种方法发起请求，RestTemplate 负责执行远程调用时，使用的都是 execute 方法，该方法定义如下代码所示：
```java
@Nullable
public <T> T execute(String url, HttpMethod method, @Nullable RequestCallback requestCallback, @Nullable ResponseExtractor<T> responseExtractor, Object... uriVariables) throws RestClientException {
    URI expanded = this.getUriTemplateHandler().expand(url, uriVariables);
    return this.doExecute(expanded, method, requestCallback, responseExtractor);
}
```
从以上代码中，发现 execute 方法首先通过 UriTemplateHandler 构建了一个 URI，然后将请求过程委托给 doExecute 方法进行处理，该方法定义如下代码所示：
```java
@Nullable
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback, @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
    Assert.notNull(url, "URI is required");
    Assert.notNull(method, "HttpMethod is required");
    ClientHttpResponse response = null;

    Object var14;
    try {
        // 创建请求对象
        ClientHttpRequest request = this.createRequest(url, method);
        if (requestCallback != null) {
            // 执行对请求的回调
            requestCallback.doWithRequest(request);
        }
        // 获取调用结果
        response = request.execute();
        // 处理调用结果
        this.handleResponse(url, method, response);
        // 使用结果提取从结果中提取数据
        var14 = responseExtractor != null ? responseExtractor.extractData(response) : null;
    } catch (IOException var12) {
        String resource = url.toString();
        String query = url.getRawQuery();
        resource = query != null ? resource.substring(0, resource.indexOf(63)) : resource;
        throw new ResourceAccessException("I/O error on " + method.name() + " request for \"" + resource + "\": " + var12.getMessage(), var12);
    } finally {
        if (response != null) {
            response.close();
        }

    }
    return var14;
}
```

## 创建请求对象

创建请求对象的入口方法如下代码所示：
```java
ClientHttpRequest request = createRequest(url, method);
```

通过跟踪上面的 createRequest 方法，发现流程执行到了前面介绍的 HttpAccessor 类。

创建 ClientHttpRequest 的过程是一种典型的工厂模式应用场景，这里直接创建了一个实现 ClientHttpRequestFactory 接口的 SimpleClientHttpRequestFactory 对象，然后再通过这个对象的 createRequest 方法创建了客户端请求对象 ClientHttpRequest 并返回给上层组件进行使用。ClientHttpRequestFactory 接口的定义如下代码所示：
```java
@FunctionalInterface
public interface ClientHttpRequestFactory {
    // 创建客户端请求对象
    ClientHttpRequest createRequest(URI var1, HttpMethod var2) throws IOException;
}

```

在 Spring 中，存在一批 ClientHttpRequestFactory 接口的实现类，而SimpleClientHttpRequestFactory 是它的默认实现，在实现自定义的 ClientHttpRequestFactory 时，开发人员也可以根据需要自行选择。

为简单起见，直接跟踪 SimpleClientHttpRequestFactory 的代码，来看它的 createRequest 方法，如下代码所示：
```java
private boolean bufferRequestBody = true;
@Override
public ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
    HttpURLConnection connection = openConnection(uri.toURL(), this.proxy);
    prepareConnection(connection, httpMethod.name());
    if (this.bufferRequestBody) {
        return new SimpleBufferingClientHttpRequest(connection, this.outputStreaming);
    } else {
        return new SimpleStreamingClientHttpRequest(connection, this.chunkSize, this.outputStreaming);
    }
}
```

在上述 createRequest 中，首先通过传入的 URI 对象构建了一个 HttpURLConnection 对象，然后对该对象进行一些预处理，最后构造并返回一个 ClientHttpRequest 实例。

通过翻阅代码，我们发现上述的 openConnection 方法只是通过 URL 对象的 openConnection 方法返回了一个 UrlConnection，而 prepareConnection 方法也只是完成了对 HttpUrlConnection 超时时间、请求方法等常见属性的设置。

在这里，注意到 bufferRequestBody 参数的值为 true，因此通过 createRequest 方法最终返回的结果是一个 SimpleBufferingClientHttpRequest 对象。

## 执行远程调用

一旦获取了请求对象，我们就可以发起远程调用并获取响应了，RestTemplate 中的入口方法如下代码所示：
```java
response = request.execute();
```

这里的 request 就是前面创建的 SimpleBufferingClientHttpRequest 类，可以先来看一下该类的类层结构，如下图所示：
![SimpleBufferingClientHttpRequest 类层结构图](https://images.happymaya.cn/assert/spring-boot/SimpleBufferingClientHttpRequest.png)

在上图的 AbstractClientHttpRequest 中，定义了如下代码所示的 execute 方法：
```java
@Override
public final ClientHttpResponse execute() throws IOException {
    assertNotExecuted();
    ClientHttpResponse result = executeInternal(this.headers);
    this.executed = true;
    return result;
}
protected abstract ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException;
```

AbstractClientHttpRequest 类的作用是防止 HTTP 请求的 Header 和 Body 被多次写入，所以在 execute 方法返回之前，我们设置了一个 executed 标志位。同时，在 execute 方法中，最终调用了一个抽象方法 executeInternal，这个方法的实现在 AbstractClientHttpRequest 的子类 AbstractBufferingClientHttpRequest 中，如下代码所示：
```java
@Override

protected ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException {
    byte[] bytes = this.bufferedOutput.toByteArray();
    if (headers.getContentLength() < 0) {
        headers.setContentLength(bytes.length);
    }
    ClientHttpResponse result = executeInternal(headers, bytes);
    this.bufferedOutput = new ByteArrayOutputStream(0);
    return result;
}
protected abstract ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput)    throws IOException;
```

和 AbstractClientHttpRequest 类一样，进一步梳理了一个抽象方法 executeInternal，这个抽象方法通过最底层的 SimpleBufferingClientHttpRequest 类实现，如下代码所示：
```java
@Override

protected ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
    addHeaders(this.connection, headers);
    // JDK <1.8 doesn't support getOutputStream with HTTP DELETE
    if (getMethod() == HttpMethod.DELETE && bufferedOutput.length == 0) {
        this.connection.setDoOutput(false);
    }
    if (this.connection.getDoOutput() && this.outputStreaming) {
        this.connection.setFixedLengthStreamingMode(bufferedOutput.length);
    }
    this.connection.connect();
    if (this.connection.getDoOutput()) {
        FileCopyUtils.copy(bufferedOutput, this.connection.getOutputStream());
    } else {
        // Immediately trigger the request in a no-output scenario as well
            this.connection.getResponseCode();
    }
    return new SimpleClientHttpResponse(this.connection);
}
```

这里通过 FileCopyUtils.copy 工具方法，我们把结果写入输出流上了，executeInternal 方法最终返回的结果是一个包装了 Connection 对象的 SimpleClientHttpResponse。

## 处理响应结果

一个 HTTP 请求处理的最后一步是从 ClientHttpResponse 中读取输入流，然后格式化为一个响应体并将其转化为业务对象，入口代码如下所示：
```java
//处理调用结果
handleResponse(url, method, response);
//使用结果提取从结果中提取数据
return (responseExtractor != null ? responseExtractor.extractData(response) : null);
```

先来看这里的 handleResponse 方法，定义如下代码所示：
```java
protected void handleResponse(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
    ResponseErrorHandler errorHandler = getErrorHandler();
    boolean hasError = errorHandler.hasError(response);
    if (logger.isDebugEnabled()) {
        try {
            logger.debug(method.name() + " request for \"" + url + "\" resulted in " + response.getRawStatusCode() + " (" + response.getStatusText() + ")" + (hasError ? "; invoking error handler" : ""));
        } catch (IOException ex) {
                // ignore
        }
        } if (hasError) {
            errorHandler.handleError(url, method, response);
    }
}
```

以上代码中，通过 getErrorHandler 方法我们获取了一个 ResponseErrorHandler，如果响应的状态码错误，我们可以调用 handleError 来处理错误并抛出异常。在这里，我们发现这段代码实际上并没有真正处理返回的数据，而只是执行了错误处理。

而获取响应数据并完成转化的工作是在 ResponseExtractor 中，该接口定义如下代码所示：
```java
public interface ResponseExtractor<T> {
    @Nullable
    T extractData(ClientHttpResponse response) throws IOException;
}
```

在 RestTemplate 类中，定义了一个 ResponseEntityResponseExtractor 内部类实现了ResponseExtractor 接口，如下代码所示：
```java
private class ResponseEntityResponseExtractor <T> implements ResponseExtractor<ResponseEntity<T>> {
    @Nullable
    private final HttpMessageConverterExtractor<T> delegate;
    
    public ResponseEntityResponseExtractor(@Nullable Type responseType) {
        if (responseType != null && Void.class != responseType) {
            this.delegate = new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
        } else {
            this.delegate = null;
        }
    }
    
    @Override
    public ResponseEntity<T> extractData(ClientHttpResponse response) throws IOException {
        if (this.delegate != null) {
            T body = this.delegate.extractData(response);
            return ResponseEntity.status(response.getRawStatusCode()).headers(response.getHeaders()).body(body);
        } else {
            return ResponseEntity.status(response.getRawStatusCode()).headers(response.getHeaders()).build();
        }
    }
}
```

在上述代码中，ResponseEntityResponseExtractor 中的 extractData 方法本质上是将数据提取部分的工作委托给了一个代理对象 delegate，而这个 delegate 的类型就是 HttpMessageConverterExtractor。

从命名上看，不难看出 HttpMessageConverterExtractor 类的内部使用 HttpMessageConverter 实现消息的转换，如下代码所示（代码做了裁剪）：
```java
public class HttpMessageConverterExtractor<T> implements ResponseExtractor<T> {
    private final List<HttpMessageConverter<?>> messageConverters;
    
    @Override
    @SuppressWarnings({"unchecked", "rawtypes", "resource"})
    public T extractData(ClientHttpResponse response) throws IOException {
        MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
        if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
            return null;
        }
        MediaType contentType = getContentType(responseWrapper);
        try {
            for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
                if (messageConverter instanceof GenericHttpMessageConverter) {
                    GenericHttpMessageConverter<?> genericMessageConverter =
                            (GenericHttpMessageConverter<?>) messageConverter;
                    if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
                        return (T) genericMessageConverter.read(this.responseType, null, responseWrapper);
                    }
                }
                if (this.responseClass != null) {
                    if (messageConverter.canRead(this.responseClass, contentType)) {
                        return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
                    }
                }
            }
        }
	…
}
```

上述方法看上去有点复杂，但核心逻辑很简单，首先遍历 HttpMessageConveter 列表，然后判断其是否能够读取数据，如果能就调用 read 方法读取数据。

最后， HttpMessageConveter 中如何实现 read 方法。

先来看 HttpMessageConveter 接口的抽象实现类 AbstractHttpMessageConverter，在它的 read 方法中我们同样定义了一个抽象方法 readInternal，如下代码所示：

```java
@Override
public final T read(Class<? extends T> clazz, HttpInputMessage inputMessage)            throws IOException, HttpMessageNotReadableException {
    return readInternal(clazz, inputMessage);
}

protected abstract T readInternal(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;
```

Spring 提供了一系列的 HttpMessageConveter 实现消息的转换，而最简单的实现方式是 StringHttpMessageConverter，该类的 read 方法如下代码所示：
```java
@Override
protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
    Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
    return StreamUtils.copyToString(inputMessage.getBody(), charset);
}
```

StringHttpMessageConverter 的实现过程：首先从输入消息 HttpInputMessage 中通过 getBody 方法获取消息体，也就是一个 ClientHttpResponse 对象，再通过 copyToString 方法从该对象中读取数据，并返回字符串结果。

至此，通过 RestTemplate 发起、执行及响应整个 HTTP 请求的完整流程就介绍完毕了。