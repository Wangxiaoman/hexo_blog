title: spring cloud eureka的初始化
date: 2016-12-29 15:31:49
tags: [架构,java,spring boot]
category: 技术
---

### 概述

一直关注spring cloud，以后做微服务必将其发扬光大，所以结合源码和网上的文章详细研究一下，本文主要是分析eureka（EurekaClient）的初始化过程。

<!-- toc -->

<!--more-->
当需要使用EurekaClient的时候，会使用EnableEurekaClient注解，那么我们就从spring boot项目中的该注解开始（EurekaServer启动过程也类似），下面是EurekaClient启动的调用链的分析。

### 1. 加载过程

#### 1.1 spring 加载过程
EnableEurekaClient -> 
EnableDiscoveryClient -> 
EnableDiscoveryClientImportSelector -> 
EnableDiscoveryClientImportSelector(selectImports方法) ->
SpringFactoriesLoader(loadFactoryNames添加spring.factories下的类，spring启动初始化这些类）

#### 1.2 spring 启动过程
spring refresh ->
annotation.ConfigurationClassParser.processImports() -> 
EurekaClientAutoConfiguration -> 
DiscoveryClient(spring boot 包) ->
CloudEurekaClient（eureka包）


### 2. EnableEurekaClient 注解

```
	@EnableDiscoveryClient
	public @interface EnableEurekaClient
```
### 3. EnableDiscoveryClient 注解

```
	Import(EnableDiscoveryClientImportSelector.class)
	public @interface EnableDiscoveryClient
```

### 4. EnableDiscoveryClientImportSelector

该类会调用父类的方法selectImports，父类中的selectImports会调用spring的loadFactoryNames,将配置文件下的类加载，先放到指定的map中等待spring初始化，实际初始化这些类是在org.springframework.context.annotation.ConfigurationClassParser.processImports()


#### 4.1 EnableDiscoveryClientImportSelector

```
	@Order(Ordered.LOWEST_PRECEDENCE - 100)
	public class EnableDiscoveryClientImportSelector
	        extends SpringFactoryImportSelector<EnableDiscoveryClient> {

	    @Override
	    public String[] selectImports(AnnotationMetadata metadata) {
	        String[] imports = super.selectImports(metadata);

	        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
	                metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

	        boolean autoRegister = attributes.getBoolean("autoRegister");

	        if (autoRegister) {
	            List<String> importsList = new ArrayList<>(Arrays.asList(imports));
	            importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
	            imports = importsList.toArray(new String[0]);
	        }

	        return imports;
	    }

	    @Override
	    protected boolean isEnabled() {
	        return new RelaxedPropertyResolver(getEnvironment()).getProperty(
	                "spring.cloud.discovery.enabled", Boolean.class, Boolean.TRUE);
	    }

	    @Override
	    protected boolean hasDefaultFactory() {
	        return true;
	    }
	}
```

#### 4.2 SpringFactoryImportSelector

EnableDiscoveryClientImportSelector的父类，调用SpringFactoriesLoader.loadFactoryNames加载配置文件下的类。

```
	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
	    if (!isEnabled()) {
	        return new String[0];
	    }
	    AnnotationAttributes attributes = AnnotationAttributes.fromMap(
	            metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

	    Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is "
	            + metadata.getClassName() + " annotated with @" + getSimpleName() + "?");

	    // Find all possible auto configuration classes, filtering duplicates
	    List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(this.annotationClass,    this.beanClassLoader)));

	    if (factories.isEmpty() && !hasDefaultFactory()) {
	        throw new IllegalStateException("Annotation @" + getSimpleName()
	                + " found, but there are no implementations. Did you forget to include a starter?");
	    }

	    if (factories.size() > 1) {
	        // there should only ever be one DiscoveryClient, but there might be more than
	        // one factory
	        log.warn("More than one implementation " + "of @" + getSimpleName()
	                + " (now relying on @Conditionals to pick one): " + factories);
	    }

	    return factories.toArray(new String[factories.size()]);
	}
```

#### 4.3 SpringFactoriesLoader

该类的方法loadFactoryNames，将FACTORIES_RESOURCE_LOCATION下的类添加到一个className的列表中。

```
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

	public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
	    String factoryClassName = factoryClass.getName();
	    try {
	        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
	                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
	        List<String> result = new ArrayList<String>();
	        while (urls.hasMoreElements()) {
	            URL url = urls.nextElement();
	            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
	            String factoryClassNames = properties.getProperty(factoryClassName);
	            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
	        }
	        return result;
	    }
	    catch (IOException ex) {
	        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
	                "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
	    }
	}
```


### 5. EurekaClient的初始化

** eureka包中resource下的spring.factories **

```
	org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
	org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
	org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
	org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
	org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration
```

EurekaClientAutoConfiguration 会初始化EurekaClientConfigBean、 EurekaInstanceConfigBean和DiscoveryClient（DiscoveryClient实际是对eurake中的DiscoveryClient的封装）

#### 5.1 EurekaClientAutoConfiguration
该类中初始化了EurekaClientConfigBean、 EurekaInstanceConfigBean和DiscoveryClient

** EurekaClientConfigBean类 **

```
	@Bean
	@ConditionalOnMissingBean(value = EurekaClientConfig.class, search = SearchStrategy.CURRENT)
	public EurekaClientConfigBean eurekaClientConfigBean() {
	    EurekaClientConfigBean client = new EurekaClientConfigBean();
	    if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
	        // We don't register during bootstrap by default, but there will be another
	        // chance later.
	        client.setRegisterWithEureka(false);
	    }
	    return client;
	}
```

** EurekaInstanceConfigBean类 **

```
	@Bean
	@ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
	public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils) {
	    EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);
	    instance.setNonSecurePort(this.nonSecurePort);
	    instance.setInstanceId(getDefaultInstanceId(this.env));

	    if (this.managementPort != this.nonSecurePort && this.managementPort != 0) {
	        if (StringUtils.hasText(this.hostname)) {
	            instance.setHostname(this.hostname);
	        }
	        RelaxedPropertyResolver relaxedPropertyResolver = new RelaxedPropertyResolver(env, "eureka.instance.");
	        String statusPageUrlPath = relaxedPropertyResolver.getProperty("statusPageUrlPath");
	        String healthCheckUrlPath = relaxedPropertyResolver.getProperty("healthCheckUrlPath");
	        if (StringUtils.hasText(statusPageUrlPath)) {
	            instance.setStatusPageUrlPath(statusPageUrlPath);
	        }
	        if (StringUtils.hasText(healthCheckUrlPath)) {
	            instance.setHealthCheckUrlPath(healthCheckUrlPath);
	        }
	        String scheme = instance.getSecurePortEnabled() ? "https" : "http";
	        instance.setStatusPageUrl(scheme + "://" + instance.getHostname() + ":"
	                + this.managementPort + instance.getStatusPageUrlPath());
	        instance.setHealthCheckUrl(scheme + "://" + instance.getHostname() + ":"
	                + this.managementPort + instance.getHealthCheckUrlPath());
	    }
	    return instance;
	}
```

** DiscoveryClient类 **

```
	@Bean
	public DiscoveryClient discoveryClient(EurekaInstanceConfig config,
	        EurekaClient client) {
	    return new EurekaDiscoveryClient(config, client);
	}
```

#### 5.2 EurekaClient的创建
在EurekaClientAutoConfiguration中的静态类EurekaClientConfiguration中，创建了CloudEurekaClient。

```
	@Bean(destroyMethod = "shutdown")
	@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
	public EurekaClient eurekaClient(ApplicationInfoManager manager,
	        EurekaClientConfig config) {
	    return new CloudEurekaClient(manager, config, this.optionalArgs,
	            this.context);
}
```

#### 5.3 CloudEurekaClient 
这个类实际继承了 DiscoveryClient（eureka-client包中的），其实spring boot中的EurekaClient最终也就是对eurekaClient（eureka-client）的封装

```
	public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
	                         EurekaClientConfig config,
	                         DiscoveryClientOptionalArgs args,
	                         ApplicationEventPublisher publisher) {
	    super(applicationInfoManager, config, args);
	    this.applicationInfoManager = applicationInfoManager;
	    this.publisher = publisher;
	    this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
	    ReflectionUtils.makeAccessible(this.eurekaTransportField);
	}
```

### 6. 实例化
spring启动的时候，调用spring的refresh（AbstractApplicationContext），fresh中会调用西面的方法来进行对imports的类进行实例化。

```
	org.springframework.context.annotation.ConfigurationClassParser.processImports()
```

参考：http://www.idouba.net/spring-cloud-source-load-eureka-client-by-annotation/


