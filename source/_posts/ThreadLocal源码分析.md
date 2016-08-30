title: ThreadLocal源码分析
date: 2016-08-29 17:41:49
tags: [线程,java]
category: 技术
---

### 概述

参考官方文档：ThreadLocal的用法，这个类提供“thread-local”变量，这些变量与线程的局部变量不同，每个线程都保存一份改变量的副本，可以通过get或者set方法访问。如果开发者希望将类的某个静态变量与线程状态关联，则可以考虑使用ThreadLocal，例如在Spring的拦截器中使用StopWatch来监控方法消耗时长。


<!--more-->

### ThreadLocal的内部结构

* ThreadLocal的静态内部类ThreadLocalMap，核心操作都是对这个map的操作

* ThreadLocalMap的entry的结构如下

		static class Entry extends WeakReference<ThreadLocal> {
	        /** The value associated with this ThreadLocal. */
	        Object value;

	        Entry(ThreadLocal k, Object v) {
	            super(k);
	            value = v;
	        }
	    }

* Thread类中有如下属性，这样ThreadLocal可以通过Thread的对象直接获取（ThradLocal和Thread都在lang包下）
	ThreadLocal.ThreadLocalMap threadLocals = null;

### 核心源码分析

#### get方法

    	public T get() {
            Thread t = Thread.currentThread();//1
            ThreadLocalMap map = getMap(t);//2
            if (map != null) {//3
                ThreadLocalMap.Entry e = map.getEntry(this);
                if (e != null)
                    return (T)e.value;
            }
            return setInitialValue();//4
        }

1.获取当前线程对象
2.获取这个对象中的ThreadLocalMap
3.如果这个map不为空，那么取值
4.如果这个map为空，那么调用setInitialValue来进行初始化

#### setInitialValue方法

    	private T setInitialValue() {
            T value = initialValue();//1
            Thread t = Thread.currentThread();//2
            ThreadLocalMap map = getMap(t);//3
            if (map != null)
                map.set(this, value);
            else
                createMap(t, value);//4
            return value;
        }

    	void createMap(Thread t, T firstValue) {//5
            t.threadLocals = new ThreadLocalMap(this, firstValue);
        }

1.调用场景为get方法获取value，但是ThreadLocalMap还没有创建，那么初始化一个null值
2.获取当前线程
3.获取ThreadLocalMap，这个过程中可能其他线程创建了该map
4.map不为空，那么set，为空那么创建map
5.创建map的方法

#### remove方法

    	public void remove() {
             ThreadLocalMap m = getMap(Thread.currentThread());//1
             if (m != null)
                 m.remove(this);//2
        }

    	ThreadLocalMap getMap(Thread t) {
            return t.threadLocals;
        }

1.根据Thread对象，获取ThreadLocalMap
2.map不为空，那么删除该key



