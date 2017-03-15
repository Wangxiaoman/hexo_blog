title: 分布式追踪-spring cloud sleuth
date: 2017-03-15 11:00:00
tags: [架构,java,分布式]
category: 技术
---

### 概述

随着现在系统的逐步分布式化、规模化、微服务的流行，系统之间的调用越来越复杂，那么有一个自上而下的追踪系统就尤为重要了，我们可以快速定位到系统的瓶颈并作出相应的调整。


<!-- toc -->

<!--more-->
### 1、 zipkin

zipkin 是一款开源的分布式实时数据追踪系统（Distributed Tracking System），基于 Google Dapper 的论文设计而来，由 Twitter 公司开发贡献。其主要功能是聚集来自各个异构系统的实时监控数据，用来追踪微服务架构下的系统延时问题。（国内开源的还有美团点评的cat、京东的hydra）

github：https://github.com/openzipkin/zipkin

Quick Start
```
	wget -O zipkin.jar 'https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec'
	java -jar zipkin.jar
```

### 2、client整合sleuth

#### 2.1 pom的配置
```
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-sleuth-stream</artifactId>
	</dependency>
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-starter-sleuth</artifactId>
	</dependency>
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
	</dependency>
```
#### 2.2 application.properties

```
	spring.zipkin.base-url=http://localhost:9411
	spring.zipkin.enabled=true

	spring.sleuth.sampler.percentage=1 - 默认0.1，测试环境100%提交
	spring.sleuth.enabled=true
	spring.sleuth.hystrix.strategy.enabled=true

	spring.rabbitmq.host=127.0.0.1
	spring.rabbitmq.port=5672
	spring.rabbitmq.username=spring
	spring.rabbitmq.password=123456
	spring.rabbitmq.virtualHost=/
```

如果按照上文的获取官网jar包方式启动了zipkin-server，那么client启动之后，服务调用的请求就会以http请求的方式打到zipkin-server端，数据存在内存中，server重启之后丢失。上面的配置都增加了rabbitmq，作为中间件，zipkin-server也按照相应的配置来通过rabbitmq接收消息。

#### 2.3 代码定义span

```
	import org.springframework.cloud.sleuth.Span;
	import org.springframework.cloud.sleuth.Tracer;

	@Autowired
	private Tracer tracer;

	Span span = this.tracer.createSpan("http:customTraceEndpoint",new AlwaysSampler());
	// do something
	this.tracer.close(span);
```

### 3、server整合sleuth
如果想定制zipkin-server的服务，比如作为服务发现节点、动态配置参数，那么可以自定义一个zipkin-server

#### 3.1 pom的配置
加入eureka（服务发现）、sleuth、rabbit（消息中间件）、mysql（驱动包）、zipkin包（或者使用zipkin-autoconfigure-ui+spring-cloud-sleuth-zipkin-stream，不过我当时使用这两包有报错，所以替换掉了）

```
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-eureka</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-sleuth-stream</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-sleuth</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-jdbc</artifactId>
	</dependency>
	<dependency>
		<groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<optional>true</optional>
	</dependency>
	<dependency>
		<groupId>io.zipkin.java</groupId>
		<artifactId>zipkin-server</artifactId>
	</dependency>
	<dependency>
	    <groupId>io.zipkin</groupId>
	    <artifactId>zipkin-ui</artifactId>
	</dependency>
```

#### 3.2 application.properties

```
	spring.rabbitmq.host=127.0.0.1
	spring.rabbitmq.port=5672
	spring.rabbitmq.username=spring
	spring.rabbitmq.password=123456
	spring.rabbitmq.virtualHost=/

	spring.datasource.schema=classpath:/mysql.sql
	spring.datasource.driver-class-name=com.mysql.jdbc.Driver
	spring.datasource.url=jdbc:mysql://localhost:3306/zipkin
	spring.datasource.username=root
	spring.datasource.password=root
	spring.datasource.initialize=true
	spring.datasource.continueOnError=true

	zipkin.storage.type=mysql -制定存储方式

```

#### 3.3 bootClass

```
	@SpringBootApplication
	@EnableEurekaClient
	@EnableZipkinStreamServer
	public class ZipkinServerApplication {

	    public static void main(String[] args) {
	        new SpringApplicationBuilder(ZipkinServerApplication.class).run(args);
	    }
	}
```

### 4、图标展示
按照上面的配置搞定之后，分别启动rabbitmq、server、client，先调用几次服务，然后查看zipkin的web站http://localhost:9411/

![查询页面](http://7xnz74.com1.z0.glb.clouddn.com/zipkin-server1.png?imageView2/2/w/1000)
![数据展示](http://7xnz74.com1.z0.glb.clouddn.com/zipkin-server2.png?imageView2/2/w/1000)


### 5、参考
http://blog.spring-cloud.io/blog/sc-sleuth.html
http://spring-cloud.io/reference/sleuth/#_3
http://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.0.9.RELEASE/


