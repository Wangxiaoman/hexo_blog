title: ClassLoader和Class.forName在加载类上的区别
date: 2016-08-29 15:54:49
tags: [java,JVM]
category: 技术
---

## 首先我们分析一下类加载的过程
class的装载大体分了三步：加载（loading）、链接（linking）、初始化（initializing）
* 加载
	1.通过类的全限定名来获取其定义的二进制字节流
	2.将字节流的代表的静态存储结构转换为方法区中运行时的数据结构
	3.在java堆中生成一个代表这个类的java.lang.Class对象，作为方法区中这些数据访问的入口
* 链接
	1.验证：文件格式的校验，元数据校验，字节码校验，符号引用的校验
	2.准备：内存分配仅包含static变量（实例变量会随着对象创建一起在heap中分配），设置的初始值只是默认值
		public static int value = 3；
	以上面赋值为例，在这个阶段，value赋值为0，而不是3。而把value赋值为3的指令putstatic发生在程序编译后，存放在类构造方法中，所以只有在该类进行初始化的时候，value才被赋值为3
* 初始化
	1.类构造器方法是由编译器自动收集类中所有类变量的赋值动作和静态语句块合并产生的，编译器的收集的顺序是按照类中的出现顺序，也就是个说静态块只能访问到类中顺序靠前的类变量
	2.类构造器对于类并不是必须的，如果没有静态块和类变量，那么不会生成
	3.虚拟机会保证类构造器在多线程的情况下加锁


-------------------


<!--more-->

## ClassLoader的loadClass方法

* java中的所有类，都需要加载到jvm中才能运行，类加载器的作用实际是把类文件中的字节码加载到内存中，jvm加载类都是通过ClassLoader的loadclass方法来加载的，ClassLoader是采用双亲委托的机制来加载类。

BootStrap ClassLoader <-> Ext ClassLoader <-> App ClassLoader

* ClassLoader中的几个内部的属性以及内部类
		// parent class loader
		private final ClassLoader parent; 

		// 注册并行加载的方法以及提供查询的接口
		private static class ParallelLoaders{} 
			- static boolean register(Class<? extends ClassLoader> c)
			- static boolean isRegistered(Class<? extends ClassLoader> c)	

		// 并行加载的注册状态，如果不是并行加载，那么该conMap为null
		private final ConcurrentHashMap<String, Object> parallelLockMap; 

		// ClassLoader的私有构造函数中，会根据是否为并行加载来进行初始化
		private ClassLoader(Void unused, ClassLoader parent) {
	        this.parent = parent;
	        if (ParallelLoaders.isRegistered(this.getClass())) {
	            parallelLockMap = new ConcurrentHashMap<>();
	            package2certs = new ConcurrentHashMap<>();
	            domains =
	                Collections.synchronizedSet(new HashSet<ProtectionDomain>());
	            assertionLock = new Object();
	        } else {
	            // no finer-grained lock; lock on the classloader instance
	            parallelLockMap = null;
	            package2certs = new Hashtable<>();
	            domains = new HashSet<>();
	            assertionLock = this;
	        }
	    }

* loadClass方法
		protected Class<?> loadClass(String name, boolean resolve)
	        throws ClassNotFoundException
	    {
	        synchronized (getClassLoadingLock(name)) {
	            // First, check if the class has already been loaded
	            Class c = findLoadedClass(name);
	            if (c == null) {
	                long t0 = System.nanoTime();
	                try {
	                    if (parent != null) {
	                        c = parent.loadClass(name, false);
	                    } else {
	                        c = findBootstrapClassOrNull(name);
	                    }
	                } catch (ClassNotFoundException e) {
	                    // ClassNotFoundException thrown if class not found
	                    // from the non-null parent class loader
	                }

	                if (c == null) {
	                    // If still not found, then invoke findClass in order
	                    // to find the class.
	                    long t1 = System.nanoTime();
	                    c = findClass(name);

	                    // this is the defining class loader; record the stats
	                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
	                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
	                    sun.misc.PerfCounter.getFindClasses().increment();
	                }
	            }
	            if (resolve) {
	                resolveClass(c);
	            }
	            return c;
	        }
	    }

    1.方法有两个参数 String name 类的名称，boolean resolve 如果该参数为true，那么会调用resolveClass(class)的方法
	2.同步块来对当前加载类进行加锁，如果不是并行加载，那么返回this，通过synchronized块进行阻塞，那么需要等到classLoader加载完当前的class，才能继续加载；如果是并行的加载用parallelLockedMap来维护className的一个锁状态，如果该名称的类已经被加载，那么会返回map中之前的锁对象，那么该类加载会阻塞

		protected Object getClassLoadingLock(String className) {
		    Object lock = this;
		    if (parallelLockMap != null) {
		        Object newLock = new Object();
		        lock = parallelLockMap.putIfAbsent(className, newLock);
		        if (lock == null) {
		            lock = newLock;
		        }
		    }
		    return lock;
		}

	3.进入同步块之后，先判断已加载的类池中是否有这个类，如果该类还没有被加载，那么会调用parent的loadclass方法进行加载直到bootstrapClassLoader进行加载；如果上面都没有成功加载到类，那么调用自己的findClass(String name)来加载，子类必须重写该类。
	4.resolve是一个boolean值，决定是否需要加载class的时候，将其进行link
	5.ClassLoader中的public方法loadClass方法是调用了loadClass(name, false)，是不会对class进行link的

-------------------------
## Class.forName

* Class.forName是调用方法Class.forName(className, true, this.getClass().getClassLoader())

* classForName中第二个参数表示loading之后是否初始化，ture的话，那么会初始化静态块，注册Driver。
com.mysql.jdbc.Driver 中的静态块
		static {
    		try {
        		java.sql.DriverManager.registerDriver(new Driver());
    		} catch (SQLException E) {
        		throw new RuntimeException("Can't register driver!");
    		}
		}
loadClass方法中的第二个参数值为false，表示还没有别link，所以无法初始化Driver中的静态块。而JDBC注册Driver的时候，用Class.forName，会经历类的创建过程，link验证、空间分配，类的初始化过程（在这个过程中，类的静态块和静态属性先回执行，然后执行构造函数）









