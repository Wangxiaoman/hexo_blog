title: Spring Ioc解读
date: 2016-11-09 11:41:49
tags: [java,spring]
category: 技术
---

# Spring Ioc

## 理论背景
我们知道在面向对象设计的软件系统中，它的底层都是由N个对象构成的，各个对象之间通过相互合作，最终实现系统地业务逻辑。

![耦合](http://7xnz74.com1.z0.glb.clouddn.com/ioc1.png?imageView2/2/w/600)

齿轮组中齿轮之间的啮合关系,与软件系统中对象之间的耦合关系非常相似。对象之间的耦合关系是无法避免的，也是必要的，这是协同工作的基础。现在，伴随着工业级应用的规模越来越庞大，对象之间的依赖关系也越来越复杂，经常会出现对象之间的多重依赖性关系。

![相互依赖](http://7xnz74.com1.z0.glb.clouddn.com/ioc2.png?imageView2/2/w/600)

**怎样解耦呢？**

![解耦](http://7xnz74.com1.z0.glb.clouddn.com/ioc3.png?imageView2/2/w/600)

***IoC***：即Inversion of Control（控制反转）的简写，它是一种设计模式，Spring只不过是实现了该模式。
IoC是工厂模式的升华，同时涉及到了反射的概念。
IoC有多种实现方法，其中，Spring是通过一种名为DI（Dependency Injection，即依赖注入）的方法实现的。

***DI***：所谓依赖注入，即组件之间的依赖关系由容器在运行期决定。
形象的来说，即由容器动态的将某种依赖关系注入到组件之中，上面所说的容器一般是通过一种配置文件进行解析（Xml等类型的文件，结合注解）。


**学习了tiny_spring(作者：黄亿华)项目和博客内容，加上自己的总结，记录一下。**

-------------------

<!--more-->

## step-1 基本容器
**问题：怎样描述一个实例？**

IoC最基本的角色有两个：容器(BeanFactory)和Bean本身。这里使用BeanDefinition来封装了bean的实例对象。

- ***BeanDefinition***
我们需要注册的bean对象，里面包含一个bean属性（Object）。

- ***BeanFactory***
里面有一个map，用来存储bean对象，以bean的名字作为key，对象实例作为value。

![step-1](http://7xnz74.com1.z0.glb.clouddn.com/ioc4.png?imageView2/2/w/600)

-------------------------

## step-2 将bean放入工厂
问题：step1中的bean是初始化好之后再set进去的，实际使用中，我们希望容器来管理bean的创建。(怎么办？)

增加一个工厂BeanFactory，对bean进行管理（同时增加beanDefinition中的属性，beanClass，beanClassName）。

- ***BeanFactory***
为了保证扩展性，我们使用Extract Interface的方法，将BeanFactory替换成接口，而使用AbstractBeanFactory和AutowireCapableBeanFactory作为其实现。BeanFactory中包含getBean和registerBeanDefinition两个方法，分别用于获取一个bean定义和注册bean

- ***AutowireCapableBeanFatory***
“AutowireCapable"的意思是“可自动装配的”，为我们后面注入属性做准备。
装配的bean工厂类会实现bean的创建方法doCreateBean，bean的对象创建完之后，会存放在beanDefinitionMap中。

![step-2](http://7xnz74.com1.z0.glb.clouddn.com/ioc5.png?imageView2/2/w/600)

-------------------------

## step-3 为bean注入属性
问题：如果需要注入的bean是包含属性的呢，那该怎么办？

- ***PropertyValue***
我们选择将属性注入信息保存成PropertyValue对象，类PropertyValues中封装了一个list，存储多个属性，并且保存到BeanDefinition中。

- ***AutowireCapableBeanFactory***
在创建bean方法的实现中，通过反射，使用Field将属性值注入。

![step-3](http://7xnz74.com1.z0.glb.clouddn.com/ioc7.png?imageView2/2/w/600)

-------------------------

## step-4 读取xml配置来初始化bean
问题：配置信息过多，过于复杂，怎样处理呢？

- ***BeanDefinitionReader***
解析配置文件，这里的BeanDefinition只是一些配置，我们还是用xml来初始化吧。我们定义了BeanDefinitionReader初始化bean，它有一个实现是XmlBeanDefinitionReader

- **解析资源**
Resource：定义资源，只提供一个返回流的接口
ResourceClassLoader：用来获取资源，Resource
BeanDefinitionReader：用来解析流，将xml中的数据解析为Map

beanFactory会根据BeanDefinitionReader解析出来的bean信息，创建实例，并注册（beanDefinitionMap）

![step-4](http://7xnz74.com1.z0.glb.clouddn.com/ioc8.png?imageView2/2/w/600)

-------------------------

## step-5 为bean注入bean
问题：基于XML文件来解析，我们已经完成，但是还有一个大问题，如果bean之间有依赖关系怎么办?

- ***BeanReference***
我们定义这个类，来表示这个属性是对另一个bean的引用。在解析的时候，根据xml中的标签将其解析，如果xml中属性的标签为ref，那么用BeanReference对其封装。

![step-5.1](http://7xnz74.com1.z0.glb.clouddn.com/ioc9.png?imageView2/2/w/600)

- ***AutowireCapableBeanFactory***
该类中的applyPropertyValues方法，会在读取xml的时候初始化，并在初始化bean的时候，进行解析和真实bean的注入。

![step-5.2](http://7xnz74.com1.z0.glb.clouddn.com/ioc10.png?imageView2/2/w/600)

- **循环依赖，A注入B，B也注入A**
我们使用lazy-init的方式，将createBean的事情放到getBean的时候才执行，getBean如下，只有bean不为null的时候，那么才调用BeanFactory的doCreateBean来创建。

![step-5.3](http://7xnz74.com1.z0.glb.clouddn.com/ioc11.png?imageView2/2/w/600)

**注意：这种循环依赖，只有单例模式下可用**
如果是多例，是没办法处理的，比如A1依赖B，B依赖A1；A2也依赖B，但是B中的依赖就没法确定是A1还是A2了。

-------------------------

## step-6 applicationContext登场
问题：使用起来很复杂，能否简化？

现在BeanFactory的功能齐全了，但是使用起来有点麻烦。
- **我们引入熟悉的ApplicationContext接口，并在AbstractApplicationContext的refresh()方法中进行bean的初始化工作。**

- ***ApplicationContext***
其实也就是封装了之前测试用例中，读取配置、初始化factoryBean、注册bean的过程.

![step-6.1](http://7xnz74.com1.z0.glb.clouddn.com/ioc12.png?imageView2/2/w/600)

-------------------------

## 总结
回头看一下
![step-6.2](http://7xnz74.com1.z0.glb.clouddn.com/ioc13.png?imageView2/2/w/600)

**我们的写代码应该怎么做？**

解耦
复用
扩展性
从小着手，逐步扩展


