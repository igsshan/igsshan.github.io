# JAVA集合类

### 集合类链路关系

![preview](https://pic1.zhimg.com/v2-c6201a2555b82d4935fbc0a58efb73a8_r.jpg)

![img](https://pic1.zhimg.com/80/v2-d8b64956fb72eb866f0dc9cabc7bd3d4_1440w.jpg)

###### 标记

```html
idea功能使用:
	使用idea查看类的继承关系图形
	https://www.cnblogs.com/deng-cc/p/6927447.html
```

### 集合类性能

##### 线程安全:

```
	Vector
	HashTable
	StringBuffer
```

##### 线程不安全

```
	HashMap
	TreeMap
	HashSet
	ArrayList
	LinkedList
```

```
list有序 set无序  map无序  query消息阻塞队列
```

### 集合类比较

#### ArrayList和Vector比较

##### 相同点

```
都是基于数组
都支持随机访问
默认容量都是10
都支持动态扩容
都支持fail-fast机制
```

###### 标注

```
" fail-fast "机制(快速失败)与其对应还有 " safe-fast "机制(失败安全)
这种机制是一种思想,它不仅仅体现在java的集合中,在我们常用的rpc框架Dubbo中,在集群容错是也有相关的实现
```

###### " fail-fast "机制(快速失败)

现象: 再用迭代器遍历一个集合对象时,如果遍历过程中对集合对象的内容进行了增加,删除,修改操作,则会抛出ConcurrentModificationException.

原理: 迭代器在遍历时直接访问集合中的内容,并且在遍历过程中使用一个 modCount变量.集合在被遍历期间如果内容发生变化,就会改变 modCount的值.每当迭代器使用    hashNext()/next()   遍历下一个元素之前,都会检测 modCount变量是够为  expectmodCount 值 ,是的话就返回遍历,否则抛出 ConcurrentModificationException 异常,终止遍历.

注意: 这里异常的抛出条件是检测到 modCount != expectmodCount 这个条件.如果集合发生变化时,修改modCount值刚好有设置了 exceptionmodCount 的值,则不会抛出异常.因此,不能依赖于这个异常是否抛出而进行并发操作的编程,这个异常只建议用于检测并发修改的bug

场景: java.util 包下的集合类都是快速失败的,不能在多线程下发生并发修改( 迭代过程中被修改 ).

###### " safe-fast "机制(失败安全)

现象: 采用失败安全的集合容器,在遍历时不是直接在集合内容上访问的,而是先复制原有集合内容,在拷贝的集合上进行遍历.

原理: 由于迭代时是对原集合的拷贝进行遍历,所以在遍历过程中对原集合所做的修改并不能被迭代器检测到,所以不会触发 ConcurrentModificationException

缺点: 基于拷贝内容的优点是避免了 ConcurrentModificationException, 但同样的,迭代器并不能访问到修改后的内容,即: 迭代器遍历的是开始遍历那一刻拿到的集合拷贝,在遍历期间原集合发生的修改迭代器是不知道的,这也就是它的缺点,同时,由于是需要拷贝的,所以比较吃内存

场景: java.util.concurrent 包下的容器都是安全失败的,可以在多线程下并发使用,并发修改.

###### 详细参考文档:  https://juejin.cn/post/6886664430063452173

##### 不同点

```
	1.Vector历史比ArrayList久远,Vector是jdk1.0 ,arrayList是jdk1.2
	2.Vector 是线程安全的 ,ArrayList是线程不安全的
	3.vector动态扩容默认是2倍 , ArrayList动态扩容是1.5倍
```

##### 存储结构

vector和ArrayList都使用 Object[] elementData保存数据,但是不同的是 ArrayList的elementData 使用了 关键字 transient 做了标记,这说明 ArrayList 的 elementData 不参与对象序列化的过程

###### 标注

```
transient : 
	一个对象只要实现了Serilizable接口，这个对象就可以被序列化，Java的这种序列化模式为开发者提供了很多便利，可以不必关系具体序列化的过程，只要这个类实现了Serilizable接口，这个的所有属性和方法都会自动序列化。但是有种情况是有些属性是不需要序列号的，所以就用到这个关键字。只需要实现Serilizable接口，将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。
	transient一般在实现了Serializable接口的类中使用。
```

##### 添加元素

vector

```java
add(E)
add(int, E)
addElement(E obj)
addAll(Collection<? extends E> c)
addAll(int, Collection<? extends E> c)
insertElementAt(E obj, int )
```

ArrayList

```java
add(E)
add(int, E)
addAll(Collection<? extends E> c)
addAll(int, Collection<? extends E> c)
```

在元素的添加上，`Vector`和`ArrayList`差不多提供了相同的接口，但是最大的不同是`Vector`提供的接口中，除了`add(int, E)`之外，都是同步接口，但是`add(int, E)`最终会调用同步方法`insertElementAt(E obj, int )`，故`Vector`添加元素都是同步方法；`ArrayList`添加元素的方法都是非同步方法。

##### 访问

vector

```java
get(int index)
elementAt(int index)
```

ArrayList

```java
get(int index)
```

在对元素的随机访问上，`Vector`比`ArrayList`多了一个`elementAt(int index)`函数，但是`elementAt(int index)`和`get(int index)`原理是一样的，故可以总结为`Vector`和`ArrayList`在随机访问元素时实现了同样的接口。最大的不同仍然是`Vector`对元素的随机访问是同步的，而`ArrayList`是非同步的。

##### 遍历

`ArrayList`提供了`foreach`, `Iterator`的遍历方式，`Vector`除此之外还提供了另外两种遍历方式：

```java
    Vector<String> sVector = new Vector<>();
    for (int i = 0 ; i < 5 ; i++) {
		sVector.add("test" + i);
	}
	
	sVector.forEach(new Consumer<String>() {
		@Override
		public void accept(String t) {
			// TODO Auto-generated method stub
			System.out.println(t);	
		}
	});
		
		
	Enumeration<String> enumeration = sVector.elements();
	while (enumeration.hasMoreElements()) {
		System.out.println(enumeration.nextElement());
	}
```

在`Vector`中，这两种方式和使用`Iterator`方式遍历最大的区别是他们不是同步的，主要原因是以上两种遍历方法不会在遍历过程中对集合中的数据进行修改。

##### 效率究竟差多少

`Vector`是线程安全的容器，它的很多方法都使用synchronzied方法进行了修饰，说明要使用Vector实例，需要先获得锁，这一过程比较耗时，但是究竟能耗时多少，是不是比`ArrayList`耗时很多？

不打算去测试在多线程环境下两者的对比，因为在使用`ArrayList`的时候，大多数场景是单线程的环境，本文就在单线程的环境中对`Vector`和`ArrayList`进行对比，这种对比不是精确的对比，只是对比一下快慢。本文从添加，遍历和随机访问三个方面进行对比。测量的方法比较简单，就是先向集合中添加元素，然后再去遍历元素，最后分别统计添加元素，遍历元素和随机访问的耗时，测试的java环境是jdk1.8.0_181。

vector

```java
	long start = System.currentTimeMillis();
	for (int i = 0 ; i < 500000 ; i++) {
		sVector.add("qiwoo_test_add" + i);
	}
	long end = System.currentTimeMillis();
	System.out.println("vector add time consuming:" + (end - start));

	Iterator<String> iterator = sVector.iterator();
	long visitStart = System.currentTimeMillis();
	while (iterator.hasNext()) {
		String str = iterator.next();
	}
	long visitEnd = System.currentTimeMillis();
	System.out.println("vector visit time consuming:" + (visitEnd -visitStart));
		
	long randAccessStart = System.currentTimeMillis();
	for (int i = 0 ; i < 500000 ; i++) {
		sVector.get(i);
	}
	long randAccessend = System.currentTimeMillis();
	System.out.println("vector random access time consuming:" +(randAccessend - randAccessStart));

```

运行结果:

vector add time consuming:95
vector visit time consuming:18
vector random access time consuming:14

ArrayList

```java
	long start = System.currentTimeMillis();
	for (int i = 0 ; i < 500000 ; i++) {
		sArray.add("qiwoo_test_add" + i);
	}
	long end = System.currentTimeMillis();
	System.out.println("array add time consuming:" + (end - start));
	
	
	
	Iterator<String> iterator = sArray.iterator();
	long visitStart = System.currentTimeMillis();
	while (iterator.hasNext()) {
		String str = iterator.next();
	}
	long visitEnd = System.currentTimeMillis();
	System.out.println("array visit time consuming:" + (visitEnd -visitStart));
	
	long randAccessStart = System.currentTimeMillis();
	for (int i = 0 ; i < 500000 ; i++) {
		sArray.get(i);
	}
	long randAccessend = System.currentTimeMillis();
	System.out.println("array random access time consuming:" +(randAccessend - randAccessStart));
```

运行结果:

array add time consuming:82
array visit time consuming:11
array random access time consuming:5

上面的结果可以发现，在单线程环境下，在元素添加和遍历上，`Vector`均比`ArrayList`慢了一些，其中添加元素慢了8%左右，遍历元素慢了64%，随机访问慢了1.8倍，这些数据可能受数据量的不同而不同，但是整体的趋势应该是一致的。 
 以上测试的时候，数据量为500000，但是实际进行Android开发的过程中产生的数据量比较少，参考下google设计容器时的数量考虑，接下来把数据量设置为1000，看下运行结果的差异

- Vector
   vector add time consuming:2
   vector visit time consuming:1
   array random access time consuming:0
- ArrayList
   array add time consuming:2
   array visit time consuming:1
   array random access time consuming:0

当数据量到1000时，`Vector`和`ArrayList`在元素的添加，遍历和随机访问上已经没有什么性能差异或者说差异很小。

##### 总结

```
ArrayList和Vector都是java中比较重要的容器，他们都可以存储各种对象，它们有相同的继承结构，提供大致相同的功能，主要的差异点如下：

Vector是线程安全的容易，可以在并发环境中安全地使用，而ArrayList是非线程安全的
ArrayList进行扩容时增加50%，Vector提供了扩容时的增量设置，但通常将容量扩大1倍
Vector可以使用Enumeration和Iterator进行元素遍历，ArrayList只提供了Iterator的方式
由于使用的线程同步，Vector的效率比ArrayList低

自java1.6之后，为了优化synchronized，java引入了偏向锁，在单线程环境中，Vector的效率已经被提高了。从刚才的对比也可以发现，在单线程环境中，数据量较少（测试数据量在100000性能差异较小）的情况下，Vector和ArrayList的性能差异已经不明显了。
```

#### ArrayList和LinkedList比较

ArrayList 底层使用的是 Obiect 数组; LinkedList 底层使用的是双向循环链表数据结构

ArrayList采用数组存储,所以插入和删除的时间复杂度受元素位置的影响,插入末尾还好,如果插入在中间,(add (int index, E element)) 时间复杂度接近 O(n) ; 而 LinkedList采用链表存储,所以插入,删除元素的时间复杂度不受元素位置的影响,都是近似 O(1) . 对于随机访问get 和 set ,ArrayList优于 LinkedList ,因为 LinkedList 要移动指针.

LinkedList不支持高效随机元素访问, 而 ArrayList 实现了 RandomAccess 接口,所以有随机访问功能.快速随机访问就是通过元素的序号可快速获取元素对象( get (int index)) 方法.所以 ArrayList随机访问快, 插入慢 ;则 LinkedList随机访问慢, 插入快.

ArrayList 的空间浪费主要体现在list 列表的结尾会预留一定的容量空间,而 LinkedList的空间浪费主要体现在它的每一个元素都需要消耗比 ArrayList 更多的空间( 因为要存放直接后继和直接前驱以及数据)

