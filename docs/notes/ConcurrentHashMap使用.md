# ConcurrentHashMap

> ConcurrentHashMap的高并发性主要取决于三个方便
>
> 1. 使用分离锁实现多线程更深层次并发访问
> 2. 使用ConcurrentHashMap的内部类HashEntry存储对象,HashEntry的不变形降低了读线程在遍历链表时加锁需求
> 3. 通过对同一个volatile变量的读写访问,协调不同线程间读写操作的内存可见性

ConcurrentHashMap默认创建包含16个Segment对象数组,每个Segment的成员对象数组table包含若干个散列表的桶,每个桶是有内部类HashEntry链接起来的一个链表

ConcurrentHashMap的get(Object key)不加锁,只有在put和remove操作才加锁,,ConcurrentHashMap将缓存变量分到多个Segment,分段加锁,每个Segment一个锁,只要多个线程不操作同一个Segment就不会产生锁争用,缺省情况下,ConcurrentHashMap会生成16个Segment,每个Segment上有一个锁,最多可以允许16个线程并发的更新而不产生锁争用

- ConcurrentHashMap中的Segment
  - Segment是一种可重入锁ReentrantLock,在ConcurrentHashMap里扮演着锁的角色,每个Segment守护者一个HsahEntry数组里的元素,当对HashEntry数组中的数据进行操作时,必须先获取它对应的Segment锁.

> 容器对比

- HashTable容器使用Synchronized保证线程安全,但是线程竞争激烈的情况下HashTable的效率会降低.因为当一个线程访问HashTable的同步方法时,其他线程访问HashTable的同步方法时,可能会进入阻塞或者轮询状态.
  - HashTable容器在竞争激烈的并发环境下表现出效率底下的原因是因为所有访问HashTable的线程都必须竞争同一把锁
- ConcurrentHashMap所使用的是分段锁机制,首先将数据分成一段一段的存储,然后给每一段数据加锁,当一个线程占用锁访问其中一个数据段的时候,其他的数据段也能被其他线程访问
  - 容器中有多把锁,每一把锁用于锁容器中的一部分数据,当多个线程访问容器里不同数据段的数据时,线程间不会存在锁竞争,从而有效的提高并发访问效率.



> ### ConcurrentHashMap的Get操作

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



Segment的get操作实现非常简单和高效,先经过一次在哈希,然后使用这个哈希值通过哈希运算定位到Segment,再通过哈希算法定位到元素

Get操作的高效之处在于整个Get操作过程中不需要加锁,除非读到的值是空的才会加锁重读,相比于HashTable,HashTable容器的Get方法是需要加锁的,那么ConcurrentHashMap的Get操作是如何做到不加锁的呢?原因在于Get方法里将要使用的共享变量都定义成Volatile. 如用于统计当前Segment大小的couut字段和用于存储值得HashEntry的value.

定义成volatile的变量,能够在线程之间保存可见性,能够被多线程同时读,并且保证不会读到过期的值,但是只能被单线程写(有一种情况可以被多线程写,就是写入的值不依赖原值),在Get操作里只需要读不需要写共享变量count和value,所以不需要加锁.之所以不会读到过期的值,是根据java内存模型的happen before 原则,对volatile字段的写入操作先于读操作,即使两个线程同时修改和获取volatile变量,get操作也能拿到最新的值,这是用volatile替换锁的经典应用场景.



> ### ConcurrentHashMap的Put操作

```java
	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```



由于put方法里需要对共享变量进行写入操作,所以为了线程安全,在操作共享变量时必须加锁.put方法首先定位到Segmetn,然后再Segment里进行插入操作.插入操作需要经历两个步骤,第一步判断是否需要对Segment里的HashEntry数组进行扩容,第二步定位添加元素的位置然后放入HashEntry数组里.

是否需要扩容.在插入元素前会先判断Segment里的HahEntry数组是否超过容量(TREEIFY_THRESHOLD = 8),如果超过阈值,数组进行扩容,值得一提的是,Segment的扩容判断比HashMap更恰当,因为HashMap在插入元素后判断元素是否已经到达容量,如果到达了就进行扩容,但是很有可能就是扩容之后没有新元素插入,这是HashMap就进行了一次无效扩容

如何扩容.扩容的时候首先会创建一个两倍于原容量的数组,然后将原数组里的元素进行再hash之后插入到新的数组里,为了高效ConcurrentHashMap不会对整个容器进行扩容,而只是对某个Segment进行扩容.



> ### ConcurrentHashMap的Size操作

如果我们要统计整个ConcurrentHashMap里元素的大小,就必须统计所有Segment里元素的大小后求和.Segment里的全局变量count是一个volatile变量,那么多线程场景下,我们是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢?不是的,虽然相加时可以获取每个Segment的count的最新值,但是拿到之后可能累加之前使用的count发生了变化,那么统计结果就不准了.所以最安全的做法,是在统计size的时候把所有Segment的put,remove和clean方法全部锁住,但是这种做法显然非常低效.

因为在累加count操作过程中,之前累加过的count发生变化的几率非常小,所以ConcurrentHashMap的做法是先尝试2次通过不锁住Segment的方式来统计各个Segment大小,如果统计的过程中,容器的count发生了变化,则采用加锁的方式来统计所有Segment的大小

那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢?使用modCount变量,在put,remove和clran方法操作元素前都会将变量modCount进行加1,那么在统计size前后比较modCount是否发生了变化,从而得知容器的大小是否发生变化