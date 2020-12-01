---
title: 第三方接口封装sdk的套路
tags: 经验
date: 2020-12-01
excerpt: 将第三方接口封装成sdk
---

整理对接第三方接口的代码，发现这个模块的代码结构很简单。

主要是两层结构：

	外部调用层（BusinessService) --> 实现调用层(HttpService)


如果封装更多一点功能，也可作为三层结构：
	
	外部调用层（BusinessService) --> 内部转换层(Service) --> 实现调用层(HttpService)


两层的结构下，大致的代码和分包是这样的：

service包，负责实现调用第三方接口

	Service1.java

		public void req1(Req1 req) {
			RestTemplate restTemplate = new RestTemplate();
	        HttpEntity<Req1> request = new HttpEntity<>(req, commonHeader());
	        ResponseEntity<Resp1> response = restTemplate.postForEntity(url, request, Resp1.class);

	        return response.getBody();
		}

model包，存放第三方接口的请求实体类，响应实体类

	Req1.java
	Resp1.java


怎么使用

	public void test() {
		resp = service1.req1(req);
	}


<b>为什么要将第三方接口的调用，封装成sdk呢？</b>

有些第三方接口提供通用的公共功能，比如发送短信功能，常常多个项目都会用到，而且接口的参数不会经常发生变化。这时，可以考虑将这类接口的调用实现封装成sdk，提供给项目使用。

<b>如何将上面的service和model封装成sdk，提供给外部调用。</b>

1.首先将上面示例的 service和model 代码封装到sdk项目
2.根据业务的需要，处理好入参和出参。
3.如果业务逻辑复杂，考虑除了上面service（调用请求的功能，暂且称为http的service）外，再向上提供一层业务层service。外部调用业务service而不直接调用http的service。示例：

service包，提供业务调用的service

	BusinessService1.java （业务用的BusinessService)

	BusinessService1 {

		private Service1 service1;

		public void busiOne(BusinessEntity1 entity1) {
			service1.req1(convertToReq1(entity1));
			service1.req2();
			service1.req3();
		}

	}

http包

	Service1.java (调用第三方接口的HttpService，这个代码文章开头已给出)


对应入参则可能需要业务相关的实体类，需要做调整，添加业务相关的实体类

model包调整为domain包和msg包。其中，domain包，提供业务用的实体类

	BusinessEntity1.java
	BusinessEntity2.java

msg包，提供第三方接口的请求实体类，响应实体类

	Req1.java
	Resp1.java

4.调用第三方接口的url，以及一些固定的入参，一般会和环境强相关。测试环境，生产环境，提供的参数会不同，因此做成配置参数，写在yml中

	* yml配置参数
	* 单独的ConfigProperty配置类加载参数
	* 对应的Service中使用对应的配置类

5.优化异常处理，作为sdk，做好自身该做的事。

