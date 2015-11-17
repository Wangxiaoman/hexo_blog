title: JVM系列知识
date: 2015-11-17 18:00:49
tags: [技术,JVM]
category: 技术
---

## 1、java字节码的结构
java的Class文件是统一的，是基于字节码（以byte为单位存储的文件）的，这样就解决了跨平台的问题，下面看一下一个Class文件的基本内容。

### Class文件头部
- magic（u4）：4个字节固定的头部验证码（给JVM识别），0xCA，0xFE，0xBA，0xBE
- minor version（u2）：JRE版本
- magor vresion（u2）：JRE版本

<!--more-->

### 常量池
- constant_pool_count（u2） 
- cp_info

### 当前类信息
- 类标示符（acess_flags）
- 该类在常量池的位置
- 该父类在常量池的位置
- 接口数量
- 接口的常量池入口（多个）

### 属性列表
- access_flag(u2) 访问标示符
- name_inex(u2) 属性名称对应常量池的入口位置
- descriptor_index(u2) 属性类型对应常量池入口位置
- attribute_count(u2) 数量
- attribute_info (attributes[attribute_count]) 属性详情

### 方法列表
- access_flag(u2) 访问标示符
- name_inex(u2) 方法名称对应常量池的入口位置
- descriptor_index(u2) 方法类型对应常量池入口位置
- attribute_count(u2) 数量
- attribute_info (attributes[attribute_count]) 方法详情


### attribute列表
- LineNumberTable，StackMapTable，以及StackMapTable相关异常信息
- 通常还有Class的attribute信息，例如SourceFile、InnerClass信息

## 字节码的加载
### ClassLoader,负责读取Class的字节流进行加载
- BootStrapClassLoader，加载java自带的核心类，是JVM内核实现的
- ExtClassLoader，加载jdk扩展类，jre/lib/ext下的jar
- AppClassLoader，加载classpath下的内容，比如项目依赖的三方包
- 自定义ClassLoader

### 加载过程
- 读取文件：找到文件，会先申请父ClassLoader来加载，如果不能加载那么由子ClassLoader来加载。没有找到对应的类，那么抛出ClassNotFoundException（会最先看这个Class是否被加载过，这个Class是否已经被加载，如果加载过直接返回实例）
- 链接：对加载的字节解析、校验，对Class对象分配内存。（如果字节码的规范不符合，那么抛出ClassNotFoundError）
- 初始化：调用Class对象自身的构造函数，对静态变量、static块进行赋值。所有类在使用前必须被初始化完毕，初始化由<clinit>保证是线程安全的。如果多个线程同时访问，必须保证static块执行完成，否则都将被阻塞。









