title: Spring注解事务配置
date: 2015-12-29 22:44:49
tags: [技术,Spring,事务]
category: 技术
---

### 概述
在springmvc上配置了注解事务，但是测试并没有生效，打开了spring的debug日志，发现调用service的时候并没有出现代理类。

<!--more-->

### 配置方法

* xml中的配置
	```xml
	<!-- 使用注解事务 -->
	<tx:annotation-driven transaction-manager="multiMybatisTransactionManager" />

	<bean id="multiMybatisTransactionManager"
	 class="org.springframework.jdbc.datasource.DataSourceTransactionManager"
	  p:dataSource-ref="multipleDataSource" />
	```

* multipleDataSource 为数据源，如果不是多数据源，用正常的datasource即可

* 在需要事务的service方法上，加上注解，就可以了
	```java
	@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
	```

### 现象

配置完之后，发现事务根本没有生效，打开了了Spring的jdbc的日志，没有发现拦截器的相关日志

### 分析
* 我的配置文件，将扫描所有包的配置放到了springmvc-servlet.xml中
	``` xml
	<context:component-scan base-package="com.blink"/>
	```
* 在web.xml中，启动springmvc-servlet.xml中我配置了
	```
	<load-on-startup>1</load-on-startup>
	```
springmvc-servlet.xml中配置了包路径下所有bean的扫描，而事务注解的配置是在另外的application*.xml下，web服务启动的时候，spring的容器会先加载springmvc-servlet.xml,然后扫描所有包，将bean初始化，包括service，我在service上加的注解还没有生效，信息就被加载了，需要事务的service不会生成带有事务属性的代理类。


### 解决
将service层的扫包和注解事务的声明放在同一个xml中，这样就可以生效了。

### 思考
* 理解框架的深层原理
* 理解spring的加载顺序和过程
* spring事务的实现原理

### 参考
http://icanfly.iteye.com/blog/778401







