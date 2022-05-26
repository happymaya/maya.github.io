---
title: 使用 RestTemplate 消费 RESTful 服务
author:
  name: superhsc
  link: https://github.com/happymaya
date: 2018-05-03 17:32:00 +0800
categories: [Spring]
tags: [SpringBoot, RestTemplate, RESTful]
---

完成 Web 服务的构建后，接下来需要做的事情就是如何对服务进行消费。

## 使用 RestTemplate 访问 HTTP 端点

RestTemplate 是 Spring 提供的用于访问 RESTful 服务的客户端的模板工具类，它位于 org.springframework.web.client 包下。

在设计上，RestTemplate 完全满足 RESTful 架构风格的设计原则。相较传统 Apache 中的 HttpClient 客户端工具类，RestTemplate 在编码的简便性以及异常的处理等方面都做了很多改进。

创建一个 RestTemplate 对象，并通过该对象所提供的大量工具方法实现对远程 HTTP 端点的高效访问，如下文。


### 创建 RestTemplate

想创建一个 RestTemplate 对象，最简单且最常见的方法是直接 new 一个该类的实例，如下代码所示：
```java
@Bean
public RestTemplate restTemplate() {
	return new RestTemplate();
}
```
这里我创建了一个 RestTemplate 实例，并通过 @Bean 注解将其注入 Spring 容器中。

通過 RestTemplate 的无参构造函数，可以看到创建它的实例时，RestTemplate 都做下面的一下事情，如下代码所示：
```java
    public RestTemplate() {
        this.messageConverters = new ArrayList();
        this.errorHandler = new DefaultResponseErrorHandler();
        this.headersExtractor = new RestTemplate.HeadersExtractor();
        this.messageConverters.add(new ByteArrayHttpMessageConverter());
        this.messageConverters.add(new StringHttpMessageConverter());
        this.messageConverters.add(new ResourceHttpMessageConverter(false));

        try {
            this.messageConverters.add(new SourceHttpMessageConverter());
        } catch (Error var2) {
        }

        this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
        if (romePresent) {
            this.messageConverters.add(new AtomFeedHttpMessageConverter());
            this.messageConverters.add(new RssChannelHttpMessageConverter());
        }

        if (jackson2XmlPresent) {
            this.messageConverters.add(new MappingJackson2XmlHttpMessageConverter());
        } else if (jaxb2Present) {
            this.messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
        }

        if (jackson2Present) {
            this.messageConverters.add(new MappingJackson2HttpMessageConverter());
        } else if (gsonPresent) {
            this.messageConverters.add(new GsonHttpMessageConverter());
        } else if (jsonbPresent) {
            this.messageConverters.add(new JsonbHttpMessageConverter());
        }

        if (jackson2SmilePresent) {
            this.messageConverters.add(new MappingJackson2SmileHttpMessageConverter());
        }

        if (jackson2CborPresent) {
            this.messageConverters.add(new MappingJackson2CborHttpMessageConverter());
        }

        this.uriTemplateHandler = initUriTemplateHandler();
    }
```

**可以清楚地看到 RestTemplate 的无参构造函数只做了一件事情，添加了一批用于实现消息转换的 HttpMessageConverter 对象。**

通过 RestTemplate 发送的请求和获取的响应，都是以 JSON 作为序列化方式，而调用后续的getForObject、exchange 等方法时所传入的参数及获取的结果都是普通的 Java 对象，我们就是通过使用 RestTemplate 中的 HttpMessageConverter 自动做了这一层转换操作。

此外， RestTemplate 还有另外一个更强大的有参构造函数，如下代码所示：
```java
public RestTemplate(ClientHttpRequestFactory requestFactory) {
    this();
    this.setRequestFactory(requestFactory);
}
```
从以上代码中，可以看到这个构造函数一方面调用了前面的无参构造函数，另一方面可以设置一个 ClientHttpRequestFactory 接口。

而基于这个 ClientHttpRequestFactory 接口的各种实现类，可以对 RestTemplate 中的行为进行精细化控制。

这方面典型的应用场景是设置 HTTP 请求的超时时间等属性，如下代码所示：
```java
@Bean
public RestTemplate customRestTemplate() {
	HttpComponentsClientHttpRequestFactory httpRequestFactory = new HttpComponentsClientHttpRequestFactory();
	httpRequestFactory.setConnectionRequestTimeout(3000);
	httpRequestFactory.setConnectTimeout(3000);
	httpRequestFactory.setReadTimeout(3000);

	return new RestTemplate(httpRequestFactory);
}
```
这里我创建了一个 HttpComponentsClientHttpRequestFactory 工厂类，它是 ClientHttpRequestFactory 接口的一个实现类。通过设置连接请求超时时间 ConnectionRequestTimeout、连接超时时间 ConnectTimeout 等属性，从而对 RestTemplate 的默认行为进行了定制化处理。

### 使用 RestTemplate 访问 Web 服务

在远程服务访问上，RestTemplate 内置了一批常用的工具方法，可以根据 HTTP 的语义以及 RESTful 的设计原则对这些工具方法进行分类，如下表所示：

| HTTP 方法 | RestTemplate 方法组                         |
| --------- | ------------------------------------------- |
| GET       | getForObject/getForEntity                   |
| POST      | postForLocation/postForObject/postForEntity |
| PUT       | put                                         |
| DELETE    | delete                                      |
| Header    | headForHeaders                              |
| 不限      | exchange/execute                            |

接下来，基于该表对 RestTemplate 中的工具方法进行练习。不过在此之前，需要先了解请求的 URL。

在一个 Web 请求中，可以通过请求路径携带参数。在使用 RestTemplate 时，也可以在它的 URL 中嵌入路径变量，示例代码如下所示：
```
("http://localhost:8082/account/{id}", 1)
```
这里我对 account-service 提供的一个端点进行了参数设置：定义了一个拥有路径变量名为 id 的 URL，实际访问时，将该变量值设置为 1。

其实，在URL 中也可以包含多个路径变量，因为 Java 支持不定长参数语法，多个路径变量的赋值可以按照参数依次设置。

如下所示的代码中，在 URL 中使用了 pageSize 和 pageIndex 这两个路径变量进行分页操作，实际访问时它们将被替换为 20 和 2。

```
("http://localhost:8082/account/{pageSize}/{pageIndex}", 20, 2)
```

而路径变量也可以通过 Map 进行赋值。如下所示的代码同样定义了拥有路径变量 pageSize 和 pageIndex 的 URL，但实际访问时，会从 uriVariables 这个 Map 对象中获取值进行替换，从而得到最终的请求路径为 `http://localhost:8082/account/20/2`。
```
Map<String, Object> uriVariables = new HashMap<>();
uriVariables.put("pageSize", 20);
uriVariables.put("pageIndex", 2);
webClient.getForObject() ("http://localhost:8082/account/{pageSize}/{pageIndex}", Account.class, uriVariables);
```

请求 URL 一旦准备好了，就可以使用 RestTemplates 所提供的一系列工具方法完成远程服务的访问。

### GET 方法组

先来看一下 get 方法组，它包括 getForObject 和 getForEntity 这两组方法，每组各有三个方法。
```java
    @Nullable
    public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
        RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
        HttpMessageConverterExtractor<T> responseExtractor = new HttpMessageConverterExtractor(responseType, this.getMessageConverters(), this.logger);
        return this.execute(url, HttpMethod.GET, requestCallback, responseExtractor, (Object[])uriVariables);
    }

    @Nullable
    public <T> T getForObject(String url, Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException {
        RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
        HttpMessageConverterExtractor<T> responseExtractor = new HttpMessageConverterExtractor(responseType, this.getMessageConverters(), this.logger);
        return this.execute(url, HttpMethod.GET, requestCallback, responseExtractor, (Map)uriVariables);
    }

    @Nullable
    public <T> T getForObject(URI url, Class<T> responseType) throws RestClientException {
        RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
        HttpMessageConverterExtractor<T> responseExtractor = new HttpMessageConverterExtractor(responseType, this.getMessageConverters(), this.logger);
        return this.execute(url, HttpMethod.GET, requestCallback, responseExtractor);
    }
```

从以上方法定义上，不难看出它们之间的区别只是传入参数的处理方式不同。

这里，注意到第三个 getForObject 方法只有两个参数（后面的两个 getForObject 方法分别支持不定参数以及一个 Map 对象），如果我们想在访问路径上添加一个参数，则需要我们构建一个独立的 URI 对象，示例如下代码所示：

例如，getForObject 方法组中的三个方法如下代码所示：
```java
String url = "http://localhost:8080/hello?name=" + URLEncoder.encode(name, "UTF-8");
URI uri = URI.create(url);
```

比如 AccountController，如下代码所示：
```java
@RestController
@RequestMapping(value = "accounts")
public class AccountController {

    @GetMapping(value = "/{accountId}")
    public Account getAccountById(@PathVariable("accountId") Long accountId) {
	    …
    }
}
```

对于上述端点，我们可以通过 getForObject 方法构建一个 HTTP 请求来获取目标 Account 对象，实现代码如下所示：
```
Account result = restTemplate.getForObject("http://localhost:8082/accounts/{accountId}", Account.class, accountId);
```
在以上代码中，可以看到 getForEntity 方法的返回值是一个 ResponseEntity 对象，在这个对象中还包含了 HTTP 消息头等信息，而 getForObject 方法返回的只是业务对象本身。这是这两个方法组的主要区别，可以根据个人需要自行选择。

### POST 方法组
**与 GET 请求相比，RestTemplate 中的 POST 请求除提供了 postForObject 和 postForEntity 方法组以外，还额外提供了一组 postForLocation 方法。**

假设有如下所示的 OrderController ，它暴露了一个用于添加 Order 的端点。

那么，通过 postForEntity 发送 POST 请求的示例如下代码所示：
```java
Order order = new Order();
order.setOrderNumber("Order0001");
order.setDeliveryAddress("DemoAddress");
ResponseEntity<Order> responseEntity = restTemplate.postForEntity("http://localhost:8082/orders", order, Order.class);
return responseEntity.getBody();
```

从以上代码中可以看到，构建了一个 Order 实体，通过 postForEntity 传递给了 OrderController 所暴露的端点，并获取了该端点的返回值。（特殊说明：postForObject 的操作方式也与此类似。）

掌握了 get 和 post 方法组后，理解 put 方法组和 delete 方法组就会非常容易了。其中 put 方法组与 post 方法组相比只是操作语义上的差别，而 delete 方法组的使用过程也和 get 方法组类似。

### exchange 方法组

**对于 RestTemplate 而言，exchange 是一个通用且统一的方法，它既能发送 GET 和 POST 请求，也能用于发送其他各种类型的请求。**
```java
    public <T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity, Class<T> responseType, Object... uriVariables) throws RestClientException {
        RequestCallback requestCallback = this.httpEntityCallback(requestEntity, responseType);
        ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
        return (ResponseEntity)nonNull(this.execute(url, method, requestCallback, responseExtractor, uriVariables));
    }
```
注意，这里的 requestEntity 变量是一个 HttpEntity 对象，它封装了请求头和请求体，而 responseType 用于指定返回数据类型。 假如前面的 OrderController 中存在一个根据订单编号 OrderNumber 获取 Order 信息的端点，那么我们使用 exchange 方法发起请求的代码就变成这样了，如下代码所示。
```java
ResponseEntity<Order> result = restTemplate.exchange("http://localhost:8082/orders/{orderNumber}", HttpMethod.GET, null, Order.class, orderNumber);
```
而比较复杂的一种使用方式是分别设置 HTTP 请求头及访问参数，并发起远程调用，示例代码如下所示：
```java
//设置 HTTP Header
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON_UTF8);

//设置访问参数
HashMap<String, Object> params = new HashMap<>();
params.put("orderNumber", orderNumber);

//设置请求 Entity
HttpEntity entity = new HttpEntity<>(params, headers);
ResponseEntity<Order> result = restTemplate.exchange(url, HttpMethod.GET, entity, Order.class);
```

## RestTemplate 其他使用技巧

实现常规的 HTTP 请求，RestTemplate 还有一些高级用法，如**指定消息转换器**、**设置拦截器**和**处理异常**等。


### 指定消息转换器
在 RestTemplate 中，实际上还存在第三个构造函数，如下代码所示：
```java
public RestTemplate(List<HttpMessageConverter<?>> messageConverters) {
    this.messageConverters = new ArrayList();
    this.errorHandler = new DefaultResponseErrorHandler();
    this.headersExtractor = new RestTemplate.HeadersExtractor();
    this.validateConverters(messageConverters);
    this.messageConverters.addAll(messageConverters);
    this.uriTemplateHandler = initUriTemplateHandler();
}
```

从以上代码中不难看出，可以通过传入一组 HttpMessageConverter 来初始化 RestTemplate，这也为消息转换器的定制提供了途径。

假如，希望把支持 Gson 的 GsonHttpMessageConverter 加载到 RestTemplate 中，就可以使用如下所示的代码。
```java
@Bean
public RestTemplate restTemplate() {
	List<HttpMessageConverter<?>> messageConverters = new ArrayList<HttpMessageConverter<?>>();
	messageConverters.add(new GsonHttpMessageConverter());
	return new RestTemplate(messageConverters);
}
```
原则上，可以根据需要实现各种自定义的 HttpMessageConverter ，并通过以上方法完成对 RestTemplate 的初始化。

### 设置拦截器

如果想对请求做一些通用拦截设置，那么可以使用拦截器。不过，使用拦截器之前，首先需要实现 ClientHttpRequestInterceptor 接口。

这方面最典型的应用场景是在 Spring Cloud 中通过 `@LoadBalanced` 注解为 RestTemplate 添加负载均衡机制。可以在 LoadBalanceAutoConfiguration 自动配置类中找到如下代码。
```java
public RestTemplateCustomizer restTemplateOnCustomizer (final LoadBalancerInterceptor loadBalanceInterceptor) {
	return restTemplate -> {List<ClientHttpRequestInterceptor> list = new ArrayList<>(
		restTemplate.getInterceptors());
		list.add(loadBalanceInterceptor);
		restTemplate.setInterceptors(list);
	};
}
```
### 处理异常

请求状态码不是返回 200 时，RestTemplate 在默认情况下会抛出异常，并中断接下来的操作，如果我们不想采用这个处理过程，那么就需要覆盖默认的 ResponseErrorHandler。示例代码结构如下所示：
```java
public void restTemplateHandlerErrot () {
	RestTemplate restTemplate = new RestTemplate();

	ResponseErrorHandler responseErrorHandler = new ResponseErrorHandler() {
		@Override
		public boolean hasError(ClientHttpResponse clientHttpResponse) throws IOException {
			return true;
		}

		@Override
		public void handleError(ClientHttpResponse clientHttpResponse) throws IOException {
			// 添加定制化的异常处理代码
		}
	};
	restTemplate.setErrorHandler(responseErrorHandler);
}
```
在上面的 handleError 方法中，可以实现任何自己想控制的异常处理代码。


## 在 spring-css 中的实现服务交互 

在其中的 customer-service 的 CustomerService 类中用于完成与 account-service 和 order-service 进行集成的 generateCustomerTicket 方法的代码结构，如下代码所示：
```java
public CustomerTicket generateCustomerTicket(Long accountId, String orderNumber) {

	logger.debug("Generate customer ticket record with account: {} and order: {}", accountId, orderNumber);

	// 创建客服工单对象
	CustomerTicket customerTicket = new CustomerTicket();

	// 从本地数据库中获取Account信息
	LocalAccount account = getAccountById(accountId);
	if (account != null) {
		return customerTicket;
	}

	// 从远程 order-service 中获取 Order 信息
	OrderMapper order = getRemoteOrderByOrderNumber(orderNumber);
	if (order == null) {
		return customerTicket;
	}
	logger.debug("Get remote order: {} is successful", orderNumber);

	// 创建并保存CustomerTicket信息
	customerTicket.setAccountId(accountId);
	customerTicket.setOrderNumber(order.getOrderNumber());
	customerTicket.setCreateTime(new Date());
	customerTicket.setDescription("TestCustomerTicket");
	customerTicketRepository.save(customerTicket);

	// 添加Metrics
	// customerTicketCounterService.counter("customerTicket.created.count");
	meterRegistry.summary("customerTickets.generated.count").record(1);
		
	return customerTicket;
}
```
这里以 getRemoteOrderByOrderNumber 方法为例，来对它的实现过程进行展开，getRemoteOrderByOrderNumber 方法定义代码如下：
```java
@Autowired
private OrderClient orderClient;

private OrderMapper getRemoteOrderByOrderNumber(String orderNumber) {
    return orderClient.getOrderByOrderNumber(orderNumber);
}
```

getRemoteAccountById 方法的实现过程也类似。

接下来构建了一个 OrderClient 类完成对 order-service 的远程访问，如下代码所示：
```java
@Component
public class OrderClient {

	private static final Logger logger = LoggerFactory.getLogger(OrderClient.class);

	@Autowired
	RestTemplate restTemplate;

	public OrderMapper getOrderByOrderNumber(String orderNumber) {

		logger.debug("Get order: {}", orderNumber);

		ResponseEntity<OrderMapper> restExchange = restTemplate.exchange(
				"http://localhost:8083/orders/{orderNumber}", HttpMethod.GET, null,
				OrderMapper.class, orderNumber);

		 OrderMapper result = restExchange.getBody();

		return result;
	}

	@Bean
	public RestTemplate restTemplate() {
		List<HttpMessageConverter<?>> messageConverters = new ArrayList<HttpMessageConverter<?>>();
		messageConverters.add(new GsonHttpMessageConverter());
		return new RestTemplate(messageConverters);
	}

	@Bean
	public RestTemplate customRestTemplate() {
		HttpComponentsClientHttpRequestFactory httpRequestFactory = new HttpComponentsClientHttpRequestFactory();
		httpRequestFactory.setConnectionRequestTimeout(3000);
		httpRequestFactory.setConnectTimeout(3000);
		httpRequestFactory.setReadTimeout(3000);

		return new RestTemplate(httpRequestFactory);
	}

	public void restTemplateHandlerErrot () {
		RestTemplate restTemplate = new RestTemplate();

		ResponseErrorHandler responseErrorHandler = new ResponseErrorHandler() {
			@Override
			public boolean hasError(ClientHttpResponse clientHttpResponse) throws IOException {
				return true;
			}

			@Override
			public void handleError(ClientHttpResponse clientHttpResponse) throws IOException {
				// 添加定制化的异常处理代码
			}
		};

		restTemplate.setErrorHandler(responseErrorHandler);
	}

}
```
注意：这里我注入了一个 RestTemplate 对象，并通过它的 exchange 方法完成对远程 order-serivce 的请求过程。且这里的返回对象是一个 OrderMapper，而不是一个 Order 对象。最后，RestTemplate 内置的 HttpMessageConverter 完成 OrderMapper 与 Order 之间的自动映射。

事实上，OrderMapper 与 Order 对象的内部字段一一对应，它们分别位于两个不同的代码工程中，为了以示区别我们才故意在命名上做了区分。

> 在使用 RestTemplate 时，如何实现对请求超时时间等控制逻辑的定制化处理？