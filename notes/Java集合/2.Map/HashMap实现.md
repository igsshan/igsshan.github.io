## 一. HashMap 的实现原理

### 1.原理

![1627525783463](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627525783463.png)

```java
HashMap 采用 Entry 数组来存储 key-value 对,每一个键值对组成了一个 Entry 实体, entry类实际上是一个单向的链表结构,它具有 Next 指针,可以链接下一个 Entry 实体.

static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
```

### 2.为什么用数组+链表?

```
数组是用来确定桶的位置,利用元素的 key 的 hash 值对数组长度取模得到

链表是用来解决 hash 冲突,当出现 hash 值一样的情形,就在数组上的对应位置形成一条链表.
```

### 3.hash 冲突还有哪些解决方法

```
大概有四种比较出名
1. 开放地址法
2. 链地址法
3. 再哈希法
4. 公共溢出区域法
```

### 4.可以用 LinkedHashMap 代替数组结构嘛?

```java
源码中结构:
Entry[] table = new Entry[capacity];
代替后:
List<Entry> table = new LinkedList<Entry>();  

是可以代替的
```

### 5.既然可以代替,为什么HashMap不用LinkedHashMap,而是用数组呢?

```
因为使用数组的效率高

在 HashMap 中,定位桶的位置是利用元素的 key 的 hash 值对数组取模得到,此时,我们已经得到桶的位置,显然数组的查找效率比 LinkedList 快.
```

##### 那ArrayList，底层也是数组，查找也快啊，为啥不用ArrayList?

```
因为采用基本数组结构，扩容机制可以自己定义，HashMap中数组扩容刚好是2的次幂，在做取模运算的效率高。

而ArrayList的扩容机制是1.5倍扩容，那ArrayList为什么是1.5倍扩容这就不在本文说明了。
```

## 二. HashMap 在什么条件下扩容

### 1.HashMap在什么条件下扩容?

```
如果 bucket 满了 (超过 load factor * current capacity) ,就要 resize.

load factor 为0.75 ,为了最大程度避免哈希冲突

current capacity 为当前数组大小
```

### 2.为什么扩容是2的次幂?

```
HashMap 为了存取高效,要尽量较少碰撞,就要尽量把数据分配均匀,每个链表长度大致相同,这个实现就再把数据存到哪个链表中的算法,这个算法实际就是取模,hash%length

但是,大家都知道这种算法不如位移运算快

因此,源码中做了优化 hash&(length-1)

也就是说 hash%length==hash&(length-1)

```

##### 那为什是2的n次方呢?

``` 
因为 2 的 n 次方 实际就是 1 后面 n 个0 , 2 的 n 次方-1 ,实际就是 n 个 1

例如长度为8时候，3&(8-1)=3 2&(8-1)=2 ，不同位置上，不碰撞。

而长度为5的时候，3&(5-1)=0 2&(5-1)=0，都在0上，出现碰撞了。

所以，保证容积是2的n次方，是为了保证在做(length-1)的时候，每一位都能&1 ，也就是和1111……1111111进行与运算。
```

### 3.为什么要先高16位异或低16位再取模运算?

jdk1.8源码:

```java
static final int hash(Object key){
	int h;
	return (key==null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```
HashMap 这么做,只是为了降低 hash 冲突的几率
```

# 三.讲讲 HashMap 的 get/put 过程

### 1.put 过程

```
对 key 的 hashCode () 做 hash运算,计算index;
如果没有碰撞直接放到 bucket 里;
如果碰撞了,以链表的形式存在 buckets 后;
如果碰撞导致链表过长(大于等于 TREEIFY_THRESHOLD),	就把链表转换成红黑树
如果及诶单已经存在就替换 old value (保证key 的唯一性)
如果 bucket 满了(超过 load factor * current capacity) 就要进行 resize
```

### 2.get过程

```
对 key 的 hashCode () 做 hash 运算,计算index;
如果在 bucket 里的第一个节点里直接命中,则直接返回;
如果有冲突,则通过 key.equals(k) 去查找对应的 Entry;
	若为树,则在树中通过 key.equals(k) 查找,时间复杂度为 O(log n)
	若为链表,则在链表中通过 key.equals(k) 查找,时间复杂度为 O(n)
```

### 3.说说 String 中 HashCode 的实现?

源码 :

```
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

```
String 类中的 hashCode 计算方法还是比较简单的，就是以31为权，每一位为字符的ASCII值进行运算，用自然溢出来等效取模。

哈希计算公式可以计为s[0]31^(n-1) + s[1]31^(n-2) + … + s[n-1]

那为什么以31为质数呢?

主要是因为31是一个奇质数，所以31*i=32*i-i=(i<<5)-i，这种位移与减法结合的计算相比一般的运算快很多。

```

# 四.为什么 HashMap 在链表元素数量超过8时改为红黑树?

### 1.知道 JDK1.8 中 HashMap 改了啥么?

```
1. 由 数组 + 链表 改为 数组 + 链表 + 红黑树
2. 优化了高位运算的 hash 算法: h^(h>>>16)
3. 扩容后,元素要么是在原位置,要么是在原位置在移动 2次幂的位置,且链表顺序不变
注意: 因为最后一条的变动, HashMap 在1.8中,不会出现死循环的问题
```

### 2.为什么在解决 hash 冲突的时候,不直接用红黑树?而选择先用链表,再转红黑树?

```
因为红黑树需要进行 左旋,右旋,变色 这些操作来保持平衡,而单链表不需要

当元素小于8 的时候,此时做查询操作,链表结构已经能保证查询性能;当元素大于8的时候,此时需要红黑树来加快查询速度,但是新增节点的效率变慢了

因此,如果一开始就用红黑树结构,元素太少,新增效率有比较慢,很浪费性能的
```

### 3.如果不用红黑树,用二叉查找树可以么?

```
可以,但是 二叉查找树在特殊情况下会变成一条线性结构(这就跟原来使用链表结构一样了),遍历查询会非常慢
```

### 4.当链表转为红黑树后,什么时候退化为链表?

```
为6的时候退化为链表.中间有个差值7可以防止链表和树之间频繁的转换.
假设一下,如果设计成链表个数超过8则就转换为树结构,链表个数小于8就树转换为链表,如果一个 HashMap 不停地插入,删除元素,链表个数在 8 左右徘徊,就会频繁的发生树转换链表,链表转换为树,效率会很低
```

# 五.HashMap 的并发问题

### HashMap 在并发编程环境下有什么问题?

```
1. 多线程扩容,引起的死循环问题
2. 多线程put的问题可能导致元素丢失
3. put非null元素后get出来的却是null
```

```
解决方案:
JDK1.8中,死循环的问题已经解决,
其他两个问题,
可以使用 ConcurrentHashMap , HashTable 等线程安全的集合类
```

# 六.一般用什么作为 HashMap 的 key?

### 1.键可以为 null 值吗?

```java\
可以, key 为 null 的时候, hash 算法最后的值以 0 来计算,也就是放在数组的第一个位置

static fianl int  hash ( Object key){
	int h;
	return (key == null) ? 0 : ( h = key.hashCode()) ^ ( h >>> 16);
}
```

### 2.你一般用什么做为 HashMap 的 key?

```
一般用 integer , String 这种不可变的类作为 HashMap 的 key ,而且 string 最常用

1. 因为字符串是不可变的,所有在它创建的时候 HashMap 就被缓存了,不需要重新计算,这就使得字符串很适合作为 Map 中的键,字符串的处理速度要快过其他的键对象,这就是 HashMap 中的键往往都是使用 String 类型的原因
2.因为获取对象的时候要用到 equals() 和 hashCode() 方法,那么键对象正确的重写这两个方法是非常重要的,这些类已经很规范的覆写了 hashCode()  已经 equals() 方法
```

### 3.我用可变类当HashMap的key有什么问题?

```java
hashcode可能发生改变，导致put进去的值，无法get出，如下所示

HashMap<List<String>, Object> changeMap = new HashMap<>();
List<String> list = new ArrayList<>();
list.add("hello");
Object objectValue = new Object();
changeMap.put(list, objectValue);
System.out.println(changeMap.get(list));
list.add("hello world");//hashcode发生了改变
System.out.println(changeMap.get(list));

```

```java
输出值如下

java.lang.Object@74a14482
null
```

### 4.如果让你实现一个自定义的class作为HashMap的key该如何实现？

```java
此题考察两个知识点
	重写hashcode和equals方法注意什么?
	如何设计一个不变类

针对问题一，记住下面四个原则即可
(1)两个对象相等，hashcode一定相等
(2)两个对象不等，hashcode不一定不等
(3)hashcode相等，两个对象不一定相等
(4)hashcode不等，两个对象一定不等

针对问题二，记住如何写一个不可变类
(1)类添加final修饰符，保证类不被继承。
如果类可以被继承会破坏类的不可变性机制，只要继承类覆盖父类的方法并且继承类可以改变成员变量值，那么一旦子类以父类的形式出现时，不能保证当前类是否可变。

(2)保证所有成员变量必须私有，并且加上final修饰
通过这种方式保证成员变量不可改变。但只做到这一步还不够，因为如果是对象成员变量有可能再外部改变其值。所以第4点弥补这个不足。

(3)不提供改变成员变量的方法，包括setter

避免通过其他接口改变成员变量的值，破坏不可变特性。

(4)通过构造器初始化所有成员，进行深拷贝(deep copy)
如果构造器传入的对象直接赋值给成员变量，还是可以通过对传入对象的修改进而导致改变内部变量的值。
例如：
public final class ImmutableDemo {  
    private final int[] myArray;  
    public ImmutableDemo(int[] array) {  
        this.myArray = array; // wrong  
    }  
}
这种方式不能保证不可变性，myArray和array指向同一块内存地址，用户可以在ImmutableDemo之外通过修改array对象的值来改变myArray内部的值。

为了保证内部的值不被修改，可以采用深度copy来创建一个新内存保存传入的值。
正确做法：
public final class MyImmutableDemo {  
    private final int[] myArray;  
    public MyImmutableDemo(int[] array) {  
        this.myArray = array.clone();   
    }   
}

(5)在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝

这种做法也是防止对象外泄，防止通过getter获得内部可变成员对象后对成员变量直接操作，导致成员变量发生改变。
```

