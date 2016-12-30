title: spring cloud eurek(二) EurekaClient的分析
date: 2016-12-29 17:25:49
tags: [架构,java,spring boot]
category: 技术
---

### 概述

接着上文的spring cloud eurake的client调用，已经了解到spring cloud中的eureka实际是对netflix的eureka包中的EurekaClient的封装，那么我们就来详细分析一下该类的实现。

<!-- toc -->

<!--more-->


### 1 类图

![相关类图](http://7xnz74.com1.z0.glb.clouddn.com/eurekaclass.png)

左边的DiscoveryClient和EurekaDiscoveryClient都是spring cloud中的类，实际上就是在EurekaDiscoveryClient中封装了EurekaClient（接口），实际调用其实就是DiscoveryClient类，所以来看一下DiscoveryClient( CloudEurekaClient类继承了DiscoveryClient)。

### 2 DiscoveryClient

下面是DiscoveryClient中的doc说明：
a) Registering the instance with Eureka Server
b) Renewal of the lease withEureka Server
c) Cancellation of the lease from Eureka Server during shutdown
d) Querying the list of services/instances registered with Eureka Server

注册、续约、去掉无效的client、服务查询

#### 2.1 构建执行计划和PoolExecutor

```
	// 执行计划
	scheduler = Executors.newScheduledThreadPool(3,
	        new ThreadFactoryBuilder()
	                .setNameFormat("DiscoveryClient-%d")
	                .setDaemon(true)
	                .build());

	// 心跳PoolExecutor
	heartbeatExecutor = new ThreadPoolExecutor(
	        1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
	        new SynchronousQueue<Runnable>(),
	        new ThreadFactoryBuilder()
	                .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
	                .setDaemon(true)
	                .build()
	);  // use direct handoff

	// 续约PoolExcutor
	cacheRefreshExecutor = new ThreadPoolExecutor(
	        1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
	        new SynchronousQueue<Runnable>(),
	        new ThreadFactoryBuilder()
	                .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
	                .setDaemon(true)
	                .build()
	);  // use direct handoff

	// 创建rest请求类
	eurekaTransport = new EurekaTransport();
	scheduleServerEndpointTask(eurekaTransport, args);

	// 初始化执行计划 见2.2
	initScheduledTasks();
```

#### 2.2 初始化执行计划（定时发送请求）

initScheduledTasks方法，指定续约线程的执行计划，由cacheRefreshExecutor，参数配置间隔时间，定时执行；心跳与其类似。

```
		if (clientConfig.shouldFetchRegistry()) {
		    // registry cache refresh timer
		    int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
		    int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
		    scheduler.schedule(
		        new TimedSupervisorTask(
		                "cacheRefresh",
		                scheduler,
		                cacheRefreshExecutor,
		                registryFetchIntervalSeconds,
		                TimeUnit.SECONDS,
		                expBackOffBound,
		                new CacheRefreshThread()
		        ),
		        registryFetchIntervalSeconds, TimeUnit.SECONDS);
		}

		if (clientConfig.shouldRegisterWithEureka()) {
		    int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
		    int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
		    logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

		    // Heartbeat timer
		    scheduler.schedule(
		            new TimedSupervisorTask(
		                    "heartbeat",
		                    scheduler,
		                    heartbeatExecutor,
		                    renewalIntervalInSecs,
		                    TimeUnit.SECONDS,
		                    expBackOffBound,
		                    new HeartbeatThread()// 见下面的心跳类
		            ),
		            renewalIntervalInSecs, TimeUnit.SECONDS);

		    // InstanceInfo replicator
		    instanceInfoReplicator = new InstanceInfoReplicator(
		            this,
		            instanceInfo,
		            clientConfig.getInstanceInfoReplicationIntervalSeconds(),
		            2); // burstSize

		    statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
		        @Override
		        public String getId() {
		            return "statusChangeListener";
		        }

		        @Override
		        public void notify(StatusChangeEvent statusChangeEvent) {
		            if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
		                    InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
		                // log at warn level if DOWN was involved
		                logger.warn("Saw local status change event {}", statusChangeEvent);
		            } else {
		                logger.info("Saw local status change event {}", statusChangeEvent);
		            }
		            instanceInfoReplicator.onDemandUpdate();
		        }
		    };

		    if (clientConfig.shouldOnDemandUpdateStatusChange()) {
		        applicationInfoManager.registerStatusChangeListener(statusChangeListener);
		    }

		    instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
		} else {
		    logger.info("Not registering with Eureka server per configuration");
		}
```

心跳线程中调用renew（http请求）续约

```
	private class HeartbeatThread implements Runnable {

	    public void run() {
	        if (renew()) {
	            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
	        }
	    }
	}
```

#### 2.3 实际调用Http请求

client获取服务列表、续约、注册（getApplications、renew、register），通过http接口和server端进行交互。代码详情请见下方，其实就是三个rest请求的封装。

```
	/**
	 * Get all applications registered with a specific eureka service.
	 *
	 * @param serviceUrl
	 *            - The string representation of the service url.
	 * @return - The registry information containing all applications.
	 */
	@Override
	public Applications getApplications(String serviceUrl) {
	    try {
	        EurekaHttpResponse<Applications> response = clientConfig.getRegistryRefreshSingleVipAddress() == null
	                ? eurekaTransport.queryClient.getApplications()
	                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress());
	        if (response.getStatusCode() == 200) {
	            logger.debug(PREFIX + appPathIdentifier + " -  refresh status: " + response.getStatusCode());
	            return response.getEntity();
	        }
	        logger.error(PREFIX + appPathIdentifier + " - was unable to refresh its cache! status = " + response.getStatusCode());
	    } catch (Throwable th) {
	        logger.error(PREFIX + appPathIdentifier + " - was unable to refresh its cache! status = " + th.getMessage(), th);
	    }
	    return null;
	}

	/**
	 * Register with the eureka service by making the appropriate REST call.
	 */
	boolean register() throws Throwable {
	    logger.info(PREFIX + appPathIdentifier + ": registering service...");
	    EurekaHttpResponse<Void> httpResponse;
	    try {
	        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
	    } catch (Exception e) {
	        logger.warn("{} - registration failed {}", PREFIX + appPathIdentifier, e.getMessage(), e);
	        throw e;
	    }
	    if (logger.isInfoEnabled()) {
	        logger.info("{} - registration status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
	    }
	    return httpResponse.getStatusCode() == 204;
	}

	/**
	 * Renew with the eureka service by making the appropriate REST call
	 */
	boolean renew() {
	    EurekaHttpResponse<InstanceInfo> httpResponse;
	    try {
	        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
	        logger.debug("{} - Heartbeat status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
	        if (httpResponse.getStatusCode() == 404) {
	            REREGISTER_COUNTER.increment();
	            logger.info("{} - Re-registering apps/{}", PREFIX + appPathIdentifier, instanceInfo.getAppName());
	            return register();
	        }
	        return httpResponse.getStatusCode() == 200;
	    } catch (Throwable e) {
	        logger.error("{} - was unable to send heartbeat!", PREFIX + appPathIdentifier, e);
	        return false;
	    }
	}
```

### 3 总结

我们可以总结一下Client服务
1）Spring cloud的Client是对eureka中DiscoveryClient的封装；
2）Client与Server通过Http请求来进行交互；
3）Client端通过ScheduleTask来进行定时调用Http的请求来保证注册、续约、和更新服务列表的功能；
4）默认的Client心跳间隔为30s，失效时长为90s。

参考：
http://didispace.com/springcloud-sourcecode-eureka/