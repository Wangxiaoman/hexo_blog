title: java容器
date: 2018-10-12 18:00:49
tags: [技术]
category: 技术
---

容器类在java语言中是非常重要的，无论是面试还是实际工作中都是重中之重，我们下面就对这个话题深入探讨一下。对于基本实现我们只做简单描述，细节大家可以直接看源码来理解，我们主要的篇幅还是放在实战上。

<!-- toc -->

### 容器概述
对于任何事物的理解，我们还是需要从定义出发，我们看一下“Thinking in java” 一书中对容器的描述：通常，我们的程序需要根据程序运行时才知道的一些标准创建新对象。若非程序正式运行，否则我们根本不知道自己到底需要多少数量的对象，甚至不知道它们的准确类型。为了满足常规编程的需要，我们要求能在任何时候、任何地点创建任意数量的对象。为解决这个非常关键的问题，Java提供了容纳对象（或者对象的句柄）的多种方式。

通过定义可以看出容器类解决的核心就是对象的容纳问题，在容器类中最重要组成是Collection和Map，我们接下来看Collection和Map是如何定义的。

------------------
<!--more-->

* **Collection**

**JDK描述**
A collection represents a group of objects, known as its elements.  Some collections allow duplicate elements and others do not.  Some are ordered and others unordered. 
**含义**
集合是用来代表一组元素，有些元素允许重复有的不允许，有些有序有些无序。
**Collection的类图如下**
![集合类图结构](/images/collection.png "集合类")

* **Map**

**JDK描述**
An object that maps keys to values.  A map cannot contain duplicate keys;each key can map to at most one value.
**含义**
存储KV映射，key为独立且只对应一个value。
**Map的类图如下**
![Map类图结构](/images/map.png "映射类")

了解完Collection和Map的定义以及类图中的各类之间的关系，我们下面来实际分析下各种容器类的特性和实现方式。

### Collection
#### List

* ArrayList：底层为数组，支持按照下标访问，访问高效。
* LinkedList：底层为双向链表，只能支持顺序访问，节点的删除和新增比较高效。
* 扩展思考：两种list特性实际就是底层数据结构的特性；两种List适用的场景，ArrayList偏重读的性能，而LinkedList更加偏重写的性能。

#### Set

* HashSet：HashMap实现
* TreeSet：TreeMap实现

#### queue

* PriorityQueue：有序列表，实现优先队列(存储对象通过继承java.util.Comparable实现)
* BlockingQueue：juc下的阻塞队列，这里就不扩展了

### Map

* HashMap：底层基于哈希表实现，支持快速查找，JDK1.8之前为数组加链表的二维结构实现，当数组hashCode冲突的时候，将值添加到链表上；JDK1.8之后有所调整，当链表长度大于阈值（默认为8），链表转化为红黑树，提高查询效率。
* TreeMap：底层为红黑树实现，有序集合。
* 核心面试点：HashMap的扩容过程，线程是否安全，以及保证线程安全的解决方案。


分析完以上容器类，我们可以看出，容器类就是一个对象仓库；而数据结构就是存储对象的规则，不同的数据结构体现了不同特性的存储和查找规则，**其实数据结构才是容器类的核心**。

在JDK中容器类体现出来实际是在JVM中的数据存储（内存中），如果是要存储到磁盘上，比如Mysql、RocksDB，那么这些存储方式是否还适用呢？（这个问题我们可以在讲解Mysql的时候再来讨论）

因为篇幅有限，各种容器类的实现细节大家可以通过阅读源码方式理解，让我们更加聚焦于面试点和项目上的实际应用。

### 实战模拟
在实际面试中，不仅仅是要看候选者对理论的掌握，更重要的是对于原理的理解以及灵活的应用，下面就是几个实际例子，逐步增加同学们对容器类的理解

#### 集合类删除问题
我们先构造一个整型的ArrayList如下：
```
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(3);
list.add(1);
list.add(1);
list.add(10);
```
然后我们来提出几个问题：
* 我们怎样删除ArrayList中的值为10的元素，下面两种方式是否有问题？
```
list.remove(10);
list.remove(new Integer(10));
```
我们来具体分析一下，ArrayList中的remove方法有两个：
E remove(int index); 
boolean remove(Object o); 
那么如果入参为int，就会调用第一个方法直接根据数组下标来删除数据，然后调用native方法System.arrayCopy将数组下面的数据上移，所以根据分析来看第一个函数会报数组越界。如果想删除List中的整型对象，就要将基本数据类型int值封装为Integer，这样会调用第二个方法。在实际项目中，在ArrayList中存储Integer元素的时候，一定要注意这个问题。

* 集合遍历过程中，怎样删除对象？
下面有几种删除方式，大家先花几分钟看看哪种方式是合理的。
```
// 1.基于数组下标循环
for (int i = 0; i < list.size(); i++) {
    if (list.get(i) == 1) {
        list.remove(1);
    }
}

// 2.foreach
list.forEach(i -> {
    if(i == 1){
        list.remove(new Integer(1));
    }
});

// 3.返回一个新列表
List<Integer> newList = new ArrayList<>(list.size());
for (int i = 0; i < list.size(); i++) {
    if (list.get(i) % 5 == 0) {
        newList.add(list.get(i));
    }
}
return newList;

// 4.迭代器
Iterator<Integer> iter = list.iterator();
while(iter.hasNext()){
    Integer element = iter.next();
    if(element == 1){
        iter.remove();
    }
}
```

如果你的答案是前两个，那么你需要对 list 的删除详细了解一下了，我们下面来逐次分析。

**第一种**方式，直接基于列表的下标删除。上面我们已经介绍了ArrayList的remove过程，这个删除方法的问题就交给大家自行分析了。（**习题1**）

**第二种**为foreach中删除，在ArrayList中有一个变量叫做modCount，对列表的更新、删除操作，都会将 modCount 加1，主要目的是为了避免对当前列表并发的新增和删除元素（无论是当前线程和其他线程）。
下面我们可以看一下foreach的代码：
```
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```
从代码中我们可以看到，进入方法后 expectedModCount 用modCount 赋值，当执行accept前后都对 modCount 和 expectedModCount 做了比较，如果不相等，那么会抛出异常，因为 accept 中调的 list 的 add、remove 方法都会将 modCount 加1，所以在删除之后两值不等，就会抛出异常。

**第三种**其实是利用副本的方式来实现，只不过多浪费了存储空间。

**第四种**迭代器的方式是比较通用的解法，ArrayList中有一个内部类Itr实现了Iterator，Iterator中维护了当前位置的索引cursor。大家可以查看源码，这里就不细聊。

#### 一键多值
**问题描述**：怎样实现一个容器类，实现一个key对应多个value?
代码示例如下：
```
    put('key',1);
    put('key',2);
    put('key',3);
    put('key',2);
    get('key'); // 结果为 [1,2,3]
```

**问题分析**：
我们通过自问自答的方式，逐步深入来分析这个问题。
1. 问：需要构造的数据存储结构属于哪类容器？ 答：KV映射（Map）。
2. 问：这个结构的value有何特点？ 答：value为一个有序不重复集合（TreeSet）。
3. 问：该结构如何设计？ 答：通过上面的分析，可以得出结论，将容器类组合起来就可以实现这个功能。

**实际解决**：
```
import java.util.HashMap;
import java.util.Map;
import java.util.TreeSet;

public class MutiHashMap<K, V> {
	private Map<K, TreeSet<V>> map = new HashMap<>();

	public TreeSet<V> get(K k) {
		return map.get(k);
	}

	public TreeSet<V> put(K k, V v) {
		TreeSet<V> set = map.getOrDefault(k, new TreeSet<V>());
		set.add(v);
		return map.put(k, set);
	}
	
	public TreeSet<V> remove(K k){
		return map.remove(k);
	}

	public static void main(String[] args) {
		MutiHashMap<String, Integer> mutiMap = new MutiHashMap<>();
		mutiMap.put("kv1", 1);
		mutiMap.put("kv1", 2);
		mutiMap.put("kv1", 3);
		mutiMap.put("kv1", 2);
		System.out.println("mutiMap kv1:" + mutiMap.get("kv1"));
	}
}

```
Guava 中的 collect 包下实现了各种类型的 MutiMap，大家可以看一下增深下理解。

#### ip库服务

**问题描述**
有一份ip文件，大概有200M以内（每一段至少覆盖100个ip），每天更新一次，格式描述如下（只截取几行）。

| 开始ip | 结束ip | 地址| 运营商|
| --- | --- | --- | --- |
| 0.0.0.0 | 0.255.255.255 |保留地址 |无 |
| 1.0.0.0 |1.0.0.255| 美国 | 亚太互联网络信息中心 |
| 1.0.1.0 |1.0.3.154 |中国福建省福州市|中国电信 |
| 1.0.3.155 |1.0.3.255 |中国福建省泉州市|中国电信 |
| 1.0.4.0 |1.0.7.255 |澳大利亚墨尔本 | Goldenit有限公司 |

需要提供一个高可用，高并发的服务，能够根据具体的ip值获取相应的地址和运营商，并且要说明该设计能提供的并发量和理由。请求接口示例如下：

```
请求接口：http://xxxx?ip=1.0.1.2
返回数据为：
{
    "address":"中国福建省",
    "operator":"中国电信"
}

```

**问题分析**

1. 需求分析：要求服务高可用，高并发。
2. 数据特征：ipv4可以转换为数字来存储；数据连续有序并且数据之前不重合；数据文件不大；每天更新一次。
3. 问题分析，大家看了需求和数据的特征，可以先自己思考一下，不妨自己有了一些思路，再来看我的分析。

**实际解决**
最简方案：将ip转化为Number，直接将数据写入mysql，使用开始ip建立索引。这样服务的并发量基本就是mysql所能承受的并发量，按照题目表数据量大概为百万级，考虑磁盘存储，查询条件又是一个范围查询，性能不会很好，个人估计也就是几百的量级。（这只是一个估测，因为机器配置和性能对其影响很大，也先不考虑数据库的缓存等因素，大家可以自己建表来实践一下）。

简单方案升级：不使用Mysql，我们将ip段的连续数据打散，都存到HashMap中，根据数据分析，基于每段至少覆盖100个ip，那么存储需要占20G(200M * 100)左右的内存。这种方式是典型的以空间换时间，内存存储加上HashMap的高效查询，支持万级QPS不是梦

继续升级：上面的方法虽然性能不错，但是资源耗费严重，有没一些其他的方案呢，这个数据连续有序的特性又要怎样利用呢？List不就是最典型的连续存储嘛，而且数据写少读多，那我们不妨就用ArrayList来试一下。
先将文件顺序写入，然后根据开始ip做排序（文件本身有序则不需要排序，保证顺序写入ArrayList即可）；ip查询的时候，我们通过二分来查找，该ip 属于[开始ip，结束ip]范围，那么返回该节点对象，否则为空。
在这种方案下，每次查询的时间复杂度是O(logN)，存储空间基本和文件体积一致，基于内存来存储，性能比上一个方法不会有太大的差距。（如果我们的数据不是每天更新一次，要提供接口实时更新，当前的结构是否还合适-**习题二**）

下面为具体的代码实现：
```
import java.util.ArrayList;
import org.springframework.util.CollectionUtils;
import lombok.Data;

public class IpList {
	private static ArrayList<Ip> ipList = new ArrayList<>();

	public static void load() {
		ipList.add(new Ip("0.0.0.0", "0.255.255.255", "保留地址", "无"));
		ipList.add(new Ip("1.0.0.0", "1.0.0.255", "美国", "亚太互联网络信息中心"));
		ipList.add(new Ip("1.0.1.0", "1.0.3.154", "中国福建省福州市","中国电信"));
		ipList.add(new Ip("1.0.3.155", "1.0.3.255", "中国福建省泉州市","中国电信"));
		ipList.add(new Ip("1.0.4.0", "1.0.7.255", "澳大利亚墨尔本","Goldenit有限公司"));
		ipList.add(new Ip("1.0.9.0", "255.255.255.255", "其他","其他"));
	}

	public static Ip search(String ip) {
		long longIp = Ip.ip2Long(ip);
		if(CollectionUtils.isEmpty(ipList)){
			return null;
		}
		int index = binarySearch(ipList, 0, ipList.size(), longIp);
		if(index >= 0){
			return ipList.get(index);
		}
		return null;
	}
	
	private static int binarySearch(ArrayList<Ip> ips, int fromIndex, int toIndex, long ip) {
		int low = fromIndex;
		int high = toIndex - 1;

		while (low <= high) {
			int mid = (low + high) >>> 1;
			Ip midIp =  ips.get(mid);
			// 该ip如果大于当前节点的结束ip，那么向后搜索
			if (midIp.getEndIp() < ip)
				low = mid + 1;
			// 该ip如果小于当前节点的开始ip，那么向前搜索
			else if (midIp.getBeginIp() > ip)
				high = mid - 1;
			else
				return mid; // key found
		}
		return -(low + 1); // key not found.
	}

	public static void main(String[] args) {
		load();
		System.out.println(search("243.12.1.1"));
		System.out.println(search("0.0.0.1"));
		System.out.println(search("1.0.1.0"));
		System.out.println(search("1.0.2.255"));
		System.out.println(search("1.0.8.0"));
	}
}

@Data
class Ip {
	public Ip(String beginIpStr, String endIpStr, String address, String operator) {
		this.beginIp = ip2Long(beginIpStr);
		this.endIp = ip2Long(endIpStr);
		this.address = address;
		this.operator = operator;
	}

	private long beginIp;
	private long endIp;
	private String address;
	private String operator;

	// ip 转换
	public static Long ip2Long(final String ip) {
		Long iplong = null;
		try {
			final String[] ipNums = ip.split("\\.");
			iplong = (Long.parseLong(ipNums[0]) << 24) + (Long.parseLong(ipNums[1]) << 16)
					+ (Long.parseLong(ipNums[2]) << 8) + (Long.parseLong(ipNums[3]));
		} catch (Exception e) {
			e.printStackTrace();
		}
		return iplong;
	}
}
```



### 总结

* ArrayList、HashMap是工作中最长用到的容器类，可以从这两个类的源码开始学习。
* 理解容器类的核心是理解各种数据结构的特点。
* 在实际容器类的使用中，目标一般是空间占得更小，读取和写入速度更快，但是往往不能够都满足，那就需要根据服务的特点做均衡。
* 死记硬背不可取，深入理解和灵活应用才是硬道理。


### 思考题

1. 在集合类删除中，分析ArrayList第一种删除方式的问题。
2. 在ip服务中，如果ip库数据会实时变化，那么ArrayList是否还合适呢？我们又应该如何处理？如果在这个前提下我们还有ip段的查询，比如ip从 0.0.0.0->3.0.0.255 那我们又该怎么处理？（提醒一下：Redis中有一种数据类型比较匹配；Guava中也有一些容器类可以利用）


