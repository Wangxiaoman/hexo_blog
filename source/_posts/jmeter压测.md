title: jmeter压测
date: 2016-10-13 14:41:49
tags: [压测]
category: 技术
---

### 概述

一直的工作环境中，都有专门的测试人员进行压测。这次代码开发完毕，自己也用下jmeter简单压测一下，测试下性能。


<!--more-->

### 一、环境
#### tomcat：
jvm参数如下：
jmap -heap thread

	Heap Configuration:
	   MinHeapFreeRatio         = 0
	   MaxHeapFreeRatio         = 100
	   MaxHeapSize              = 4164943872 (3972.0MB)
	   NewSize                  = 87031808 (83.0MB)
	   MaxNewSize               = 1388314624 (1324.0MB)
	   OldSize                  = 175112192 (167.0MB)
	   NewRatio                 = 2
	   SurvivorRatio            = 8
	   MetaspaceSize            = 21807104 (20.796875MB)
	   CompressedClassSpaceSize = 1073741824 (1024.0MB)
	   MaxMetaspaceSize         = 17592186044415 MB
	   G1HeapRegionSize         = 0 (0.0MB)

#### mysql：
库里的数据量比较小，基本都在万条以下，mysql基本无压力

### 二、压测案例

#### 压测工具以及配置
		工具：jmeter
		Thread group：yes
		thread：50个
		loop：100

#### 压测接口
* 因为不想干扰测试环境的数据，只压了查询接口

#### 压测结果
* 大部分接口的qps在300+
* 但是其中有一个接口qps总是在25左右，但是从服务端的接口处理时间日志上看，处理时长只有30ms-50ms左右

#### 结果分析
* 从这里来看，压测结果显然是不正常的，就处理时长来看，单线程的顺序处理也可以有至少20的qps
* 详细分析该接口，该接口返回一个相对较大的json字符串，大概有200k-400k左右
* 看到这里，初步确定就是这里的问题了，于是先查测试服务器的带宽（运维确认是上、下行不限制），然后查询本地的带宽大概是20M左右，整个公司都在使用，如果按照25*400=10M，应该也差不多

#### 确认分析结果
于是把这个方法的返回数据集缩短（原有的处理逻辑不变），果然qps就到了300左右

### 三、结论
做压测的时候，一定需要考虑自身的网络情况，qps实际上会受到了测试环境、压测本机网络的限制，需要综合考虑



