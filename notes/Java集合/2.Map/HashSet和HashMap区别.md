# HashSet 和 HashMap 的区别

```
HashSet 底层是基于 HashMap 实现的,只不过 HashSet 里面的 HashMap 所有的 value 都是同一个 Object 而已,因此 HashSet 也是非线程安全的
```



| HashMap                                                 | HashSet                                                      |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| 实现了Map接口                                           | 实现了set接口                                                |
| 存储键值对                                              | 仅存储对象                                                   |
| HashMap使用键(key)计算HashCode                          | HashSet使用成员对象计算hashCode值,对于两个对象来说hashCode可能相同,所以equals()方法用来判断对象的相等性,如果两个对象不同的话,那么返回false |
| HashMap相对于HastSet比较快,因为它是使用唯一的键获取对象 | HashSet较HashMap来说比较慢                                   |



