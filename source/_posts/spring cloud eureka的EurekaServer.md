title: spring cloud eureka（三） EurekaServer
date: 2016-12-30 14:05:49
tags: [架构,java,spring boot]
category: 技术
---

### 概述
本文继续分析eureka中的EurekaServer，下图展示了client、server之间的交互关系。
![交互图](/images/eurekaServer.png)

- Service Provider会向Eureka Server做Register（服务注册）、Renew（服务续约）、Cancel（服务下线）等操作。
- Eureka Server之间会做注册服务的同步，从而保证状态一致
- Service Consumer会向Eureka Server获取注册服务列表，并消费服务
- 代码分析(相关类：eurake-core 包中的ApplicationResource、PeerAwareInstanceRegistry、AbstractInstanceRegistry)

<!-- toc -->

<!--more-->


![注册方法时序图](/images/eurekaServerRegister.png)


### 1. 实例的注册入口(addInstance方法)

ApplicationResource类

```
	private final PeerAwareInstanceRegistry registry;

	@POST
	@Consumes({"application/json", "application/xml"})
	public Response addInstance(InstanceInfo info,
	                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
	    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
	    // validate that the instanceinfo contains all the necessary required fields
	    .......
	    // handle cases where clients may be registering with bad DataCenterInfo with missing data
	    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
	    if (dataCenterInfo instanceof UniqueIdentifier) {
	        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
	        if (isBlank(dataCenterInfoId)) {
	            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
	            if (experimental) {
	                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
	                return Response.status(400).entity(entity).build();
	            } else if (dataCenterInfo instanceof AmazonInfo) {
	                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
	                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
	                if (effectiveId == null) {
	                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
	                }
	            } else {
	                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
	            }
	        }
	    }

	    registry.register(info, "true".equals(isReplication));
	    return Response.status(204).build();  // 204 to be backwards compatible
	}
```

### 2. 注册实例信息
PeerAwareInstanceRegistry类

```
	@Override
    public void register(final InstanceInfo info, final boolean isReplication) {
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        super.register(info, leaseDuration, isReplication);
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
```

AbstractInstanceRegistry类（register方法）
```
	public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
			// 1、判断这个实例之前是否存在ConcurrentHashMap中，如果不存在，则将数据添加
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                // 如果这段时间有其他server注册了实例，并将实例信息同步过来，那么
              // gMap可能不为null，所以增加了一个判断
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
           // 如果存在，获取之前的gMap
           Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());

            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfRenewsPerMin > 0) {
                        // Since the client wants to cancel it, reduce the threshold
                        // (1
                        // for 30 seconds, 2 for a minute)
                        this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
                        this.numberOfRenewsPerMinThreshold =
                                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }

            //2、 针对实例，创建一个租户类
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            // 3、将实例key-id，value-租户放到ConMap中
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
           // 4、设置租户的服务开始时间
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
           // 5、设置该实例的缓存失效
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```

### 3. 同步信息到其他Server节点
replicateToPeers方法（peerEurekaNodes，server启动就会注册到Node），该方法会同步所有信息给其他PeerEurekaNode，在PeerEurekaNode中，封装了http的请求（canel，heart，register，update，delete）。

```
	private void replicateToPeers(Action action, String appName, String id,
	                          InstanceInfo info /* optional */,
	                          InstanceStatus newStatus /* optional */, boolean isReplication) {
	Stopwatch tracer = action.getTimer().start();
	try {
	    if (isReplication) {
	        numberOfReplicationsLastMin.increment();
	    }
	    // If it is a replication already, do not replicate again as this will create a poison replication
	    if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
	        return;
	    }

	    for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
	        // If the url represents this host, do not replicate to yourself.
	        if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
	            continue;
	        }
	        replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
	    }
	} finally {
	    tracer.stop();
	}
```

```
	private void replicateInstanceActionsToPeers(Action action, String appName,
	                                             String id, InstanceInfo info, InstanceStatus newStatus,
	                                             PeerEurekaNode node) {
	    try {
	        InstanceInfo infoFromRegistry = null;
	        CurrentRequestVersion.set(Version.V2);
	        switch (action) {
	            case Cancel:
	                node.cancel(appName, id);
	                break;
	            case Heartbeat:
	                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
	                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
	                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
	                break;
	            case Register:
	                node.register(info);
	                break;
	            case StatusUpdate:
	                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
	                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
	                break;
	            case DeleteStatusOverride:
	                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
	                node.deleteStatusOverride(appName, id, infoFromRegistry);
	                break;
	        }
	    } catch (Throwable t) {
	        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
	    }
	}
```

### 4. 其他消息同步
PeerAwareInstanceRegistry中的cancel（shutdown）、renew（heartbeat）和上面的register类似，都是调用父类AbstractInstanceRegistry的实现，然后广播到其他节点。

```
	public boolean renew(final String appName, final String id, final boolean isReplication) {
	    if (super.renew(appName, id, isReplication)) {
	        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
	        return true;
	    }
	    return false;
	}

```

参考：
http://nobodyiam.com/2016/06/25/dive-into-eureka/







