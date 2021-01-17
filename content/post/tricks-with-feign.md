---
title: "使用 spring cloud openfeign 的一些小技巧"
date: 2021-01-17T15:10:12+08:00
lastMod: 2021-01-17T15:10:12+08:00
tags: [spring cloud openfeign, 分享, java, spring]
comments: true
---

spring cloud openfeign（以下简称 feign） 通过一个额外定义的 interface 文件作为接口定义，可以将对外提供的 HTTP 接口转换为 API 接口，提供方和调用方需要共同依赖接口文件，将隐式的依赖关系显性表示出来。而且在这个接口文件上也可以大作文章，比如配置服务发现、接口拦截操作等。

一个最简单的 feign 接口文件 `DemoClient.java`：

```java
package com.example.demo;

@FeignClient(name="demo", url="http://127.0.0.1:8081/")
public interface DemoClient {

	@GetMapping("/hello")
	String hello(@RequestParam String name);
}
```

`name` 为全局唯一，是这个 FeignClient 的唯一标识，`url` 为提供方的接口地址。理论上 FeignClient 文件由接口提供方作为合约文件给到调用方，但是即使提供方未提供，只要提供方暴露了 HTTP 接口，那么调用方就可以通过定义 FeignClient 文件将 HTTP 接口调用转换为 API 调用。

调用方使用 DemoClient 示例：

```java
package com.example.demo;

@Service
public class DemoService {

	@Autowired
	private DemoClient client;
	
	public String getHelloMessage() {
		String name = "kyon";
		String msg = client.hello(name);
		return msg;
	}
}
```

怎么样，是不是完全不会意识到 `client.hello(name)` 居然是在调用 HTTP 接口！

## 配置服务发现

除了通过 url 获取到调用方的地址，在微服务以及 k8s 如此横行的当下，提供方服务随时会更换 ip，那么引入服务发现机制就很有必要了，这样只要服务名称不变，就可以拿到提供方地址，而不需要再指定 url。更进一步地说，如果提供方有水平部署了多个，feign 还（通过 ribbon）顺便帮忙做了负载。

假设提供方通过引入依赖包 `spring-cloud-starter-zookeeper-discovery` 开启了基于 zookeeper 的服务发现，那么如下所示将 `DemoClient.java` 中的 `name` 设置为提供方的应用名(假设 `spring.application.name=demo-app`)，就可以通过服务发现机制自动获得提供方地址，不再需要配置 `url` 属性（如果配置了 `url` 属性的话，就会覆盖掉服务发现提供的地址）：

```java
package com.example.demo;

@FeignClient(name="demo-app")
public interface DemoClient {
	@GetMapping("/hello")
	String hello(@RequestParam String name);
}
```

如果提供方需要提供不同的 FeignCliet, 该如何区分呢？可以通过设置不同的 ContextId 做到，如下所示：

```java
/*DemoClient1.java*/
package com.example.demo;

@FeignClient(name="demo-app", contextId="demo1")
public interface DemoClient1 {
	@GetMapping("/hello")
	String hello(@RequestParam String name);
}

/*DemoClient2.java*/
package com.example.demo;

@FeignClient(name="demo-app", contextId="demo2")
public interface DemoClient2 {
	@GetMapping("/hello")
	String hello(@RequestParam String name);
}
```

目前这边的使用场景是，对于不同的调用方，提供不同的 client 进行隔离。

## 配置 FeignClient

有两种配置方式，一种是在配置文件比如 properties 文件中配置，另外一种是通过类文件配置。最开始我是倾向于配置文件中单独配置，因为比较简洁，但是后来遇到不同版本代码共用一个配置中心导致的问题，现在倾向于在代码中通过类文件配置。

### 通过配置文件配置

首先，需要定义好自定义类比如设置登录校验头的 `CustomRequestInterceptor.java`：

```java
package com.example.demo;

public class CustomRequestInterceptor implements RequestInterceptor {  

	@Override  
	public void apply(RequestTemplate template) {
		String headerValue = "Bearer-token:xxx";  
		template.header("Authorization", headerValue);  
	}  
}
```

配置方式

```properties
## 对于所有 feign client 生效
feign.client.config.default.read-timeout=60000

## 仅对于 demo-app feign client 生效
feign.client.config.demo-app.requestInterceptors=com.example.demo.CustomRequestInterceptor
```

除了 `requestInterceptors` 外，feign 还有各种配置项，比如 decoder 等，可以自行上网搜索。

### 通过类文件配置

新增一个配置文件：

```java
package com.example.demo;

public class FeignConfig {
	@Bean  
	public RequestInterceptor interceptor() {  
		return new CustomRequestInterceptor();  
	}
}
```

把 demo-app 的 feign client 拷贝过来，并加上 config 配置：

```java
package com.example.demo;

@FeignClient(name="demo-app", configuration = FeignConfig.class)
public interface DemoClient {
	@GetMapping("/hello")
	String hello(@RequestParam String name);
}
```

这就成了。

## 打开调试日志

将第三方提供的 HTTP 接口通过 feign client 方式转换为 API 接口进行调用时，比较常见的问题是，可能参数没有传对或者什么，导致接口调用报错，但是 HTTP 原始的报错被 feign 给吞掉了，给调试带来了麻烦，这时候，只要把 feign 的 debug 日志打开，这个问题就迎刃而解了！打开方式如下：

```properites
feign.client.config.demo-app.loggerLevel=FULL

logging.level.com.example.demo.DemoClient=DEBUG
```

## 总结

目前来说，这边认为 feign 使用起来还是比较顺手的，可能多了一步 FeignClient 接口定义，但是考虑到在调用时省下的代码，强制表明依赖，支持服务发现以及负载均衡等好处，还是可以回本的，而且可以自定义接口拦截，这样即使有登录验证也没问题了。而调试的时候可以打开完整的 trace 日志，可以看到清晰的 HTTP 交谈信息，不会因为包了一层而变得不透明。日常使用起来足够了。
