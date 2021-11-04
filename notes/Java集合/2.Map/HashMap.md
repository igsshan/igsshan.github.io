# HashMap

## 数据结构

HashMap采用的是拉链法解决哈希冲突

HashMap的实现不是同步的,是线程不安全的,允许键（只允许存在一个null key）,值为null, 不保证有序(比如插入的顺序),也不保证顺序不随时间变化(哈希表加倍扩容后,数据会有迁移)

JDK1.8之前的HashMap是由 数组+ 链表 组成,数组是HashMap的主体,链表则是主要为了解决哈希冲突( 两个对象调用HashCode 方法计算的哈希值经哈希函数算出来的地址被别的元素占用) 而存在的( "拉链法" 解决冲突)
JDK1.8之后在解决哈希冲突时有了较大的变化.当链表长度阈值大于8并且当前数组的长度大于64时,此时索引位置上的所有数据改为使用红黑树存储.

![1627437675539](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627437675539.png)

```
1. 使用哈希表( 散列表 )来进行数据存储,并使用链地址法来解决哈希冲突
2. 当链表长度大于等于8时,并且当前数组的长度大于64时,将链表转换成红黑树来存储
3. 每次进行二次幂的扩容,即扩容为原容量的两倍
```

```
补充:将链表转换成红黑树前会判断,即使阈值大于8,但是数组的长度小于64,此时并不会将链表变为红黑树,而是选择进行数组扩容.
	这样做的目的是因为数组比较小,尽量避开红黑树结构,这种情况下变为红黑树结构,反而会降低效率,因为红黑树需要进行左旋,右旋,变色这些操作来保持平衡.同时数组长度小于64时,搜索时间相对要快些,所以综上所述为了提高性能和减少搜索时间,底层阈值大于8并且长度大于64是,链表才会转换成红黑树,具体可以参考treeifyBin()方法.
	当然,虽然增加 了红黑树做为底层数据结构,结构变复杂了,但是 阈值大于8并且数组长度大于64时,效率也变得更高效
```

![1627443300523](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627443300523.png) 

![1627454161907](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627454161907.png)

```
HashMap 中有几个重要的成员变量, table,size,threshold,loadFactor,modCount
```

```
	1.Node<K,V>[] table : 存储数据的哈希表;初始长度length =16(DEFAULT_INITIAL_CAPACITY),扩容时容量为原先的两倍(n*2)
	2.final float loadFactor : 负载因子,确定数组长度与当前所能存储的键值对最大值的关系;不建议轻易修改,除非情况特殊
	3.int threshold : 所能容纳的 key-value对的极限;threshold=length *load factor ,当存在的键值对大于该值,则进行扩容
	4.transient int modCount : HashMap 结构修改次数(例如每次put新值时则自增1)
	5.int size : 当前 key-value 个数
```

### put元素



![1627522349773](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627522349773.png)

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

      final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

  //代码太多占篇幅，省略掉resize() ;

```

分析:

```
1.从源码中可以看到调用put,实际上就是调用了putVal方法,它会将 key 进行一次 Hash 计算一次,计算出来的值就是这个 key 在 Node 数组中的索引,所以在进行 get 操作的时候会通过这个索引找到相应的键值,时间复杂度为 O(1) 
,下面再来看看 putVal的操作
2. 这段意思是如果这个数组为空那么就把这个数组 resize 一下,简单的概括一下 resize(),如果 Node 数组为空那么就把它初始化为一个负载因子为 0.75(默认),长度为 16 (默认) 的数组,否则,就将当前数组增长为以前数组的两倍.注意:如果数组长度为 16 ,但是它的门限值只有 16*0.75 = 12(门限值 = 初始长度* 负载因子),只要数组中元素超过门限值就会进行 resize ,扩容两倍.
```

```java
if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
```

```java
3. 通过对 hash 值和长度 -1 进行按位与作为索引查找,如果这个位置没有值,就生成一个新的节点插入

        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
```

```java
4. 观察这个else 中的逻辑,如果这个位置以前有值
5. 就看这个值的hash值还有 key 值等不等于以前的值,如果都等,那么说明 key 是一样的,就会直接返回以前的那个 node => p ,来对旧节点进行操作

            if (p.hash == hash &&
              ((k = p.key) == key || (key != null && key.equals(k))))
              e = p;

```

```java
6. 如果 key 值不等,就说明存在 hash 碰撞(就是 hash 值相等, key 值不相等),那么接下来就看看当前节点是不是树节点(红黑树),如果是树节点就进行树节点 put 操作

            else if (p instanceof TreeNode)
              e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

```java
7. 如果 key 值不等 ,并且不是树节点,那么说明现在存入的是链表,就循环这个链表,如果遇到节点为空,就将节点插入,如果插入后的节点数量超过8,那么就会将这个链表进行树化. 如果再循环链表过程中又遇到有相同的 key ,就直接对旧节点返回

          else {
              for (int binCount = 0; ; ++binCount) {
                  if ((e = p.next) == null) {
                      p.next = newNode(hash, key, value, null);
                      if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                          treeifyBin(tab, hash);
                      break;
                  }
                  if (e.hash == hash &&
                      ((k = e.key) == key || (key != null && key.equals(k))))
                      break;
                  p = e;
              }
          }

```

```java
8. 经过一波操作,终于是拿到了需要插入节点在数组中的位置,但这个时候还需要看看,这个位置是不是已经存在数据了,如果存在就把以前的那个数据返给你,如果为空就说明可以插入你想要插入的数据了.我们已经获得了插入数据的位置,这个时候这个位置可能为空,可能不为空.

            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
```

```
9. put 操作的最后,有个 modCount ,它是记录数据被修改的次数,为什么需要这个次数的存在,简单提一下,因为 HashMap 使用迭代器但是其他线程修改了这个 HashMap ,就会有 ConcurrentModificationException 异常, 这个异常的抛出就是因为,在生成 HashMap 迭代器的时候会将这个修改次数赋值给迭代器,当其他线程又去修改 HashMap 就会造成数据的不一致,所以使用这个修改次数就是一个另类的线程安全,即 fail-fast 策略
```

```java
10. 接下来看看size (指当前 HashMap 的 size),因为要插入节点到数组中所以先自增,如果超过门限值,数组就翻倍. afterNodeInsertion(evict) ,这个在网上查询一下,是为了继承 HashMap 的 LinkedHashMap 类服务的, LinkedHashMap 中被覆盖的 afterNodeInsert方法 ,用来回调移除最早放入 Map 的对象, 至此 put操作结束

  ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;

```



### get元素

![1627522432460](C:\Users\igsshan\AppData\Roaming\Typora\typora-user-images\1627522432460.png)

源码 :

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
      final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

分析 :

```java
1. 同样 get 操作是通过调用 getNode 方法来实现的, 首先对 key 进行 hash计算, 传入函数,返回 getNode 函数返回的值.
2. 第一个 if 判断,如果 table 为空就返回空值

    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {

```

```java
3. 不为空就先去看看第一个位置的节点 hash 值和 key 值是否相同( 为什么要去先看第一个,因为存在hash碰撞的几率比较小,通常链表中的节点数为一个,没必要去循环遍历整个链表,直接先看看第一个节点就是了,这里是为了效率考虑)

             if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;

```

```java
4. 当然如果链表中不止一个节点那么就需要循环遍历了,如果存在多个 hash 碰撞,这个是跑不掉的,如果节点是树节点那么就使用树节点的 get 方法来取数就是了,到这 get 也结束了

             if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }

```

