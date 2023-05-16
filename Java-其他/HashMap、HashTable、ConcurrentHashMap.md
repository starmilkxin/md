# HashMap

HashMap 底层数据结构

- HashMap 在 jdk1.7 及以前为数组+链表的形式，在 jdk1.8 及以后为数组+链表+红黑树实现。

HashMap 扩容

- 当超出负载因子计算的阈值后会扩容。
- 当试图转化为红黑树但数组长度小于64时会扩容。
- 扩容最多扩容到 Integer 的最大值。

在扩容时，首先申请新的空间，之后再将旧的 table 中的值重新 hash 到新 table 中，所以此时访问 hashmap 时，会因为还未及时迁移导致得到 value 为 null。

```java
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
```

Hashmap 转化为红黑树

- HashMap put时，如果 key 对应的 hash 桶不为空且第一个不是要找的 key，而且哈希桶为链表，那么则会遍历该链表插入最后，为的是计算链表长度。
- 如果链表长度>=8时，则试图将其转换为红黑树，此时如果数组长度小于64，则会选择扩容。
- 转化为红黑树的阈值为8，退化为链表的阈值为6。
- 红黑树的阈值为8是因为key的分布通常服从泊松分布，在8的时候的概率很小，所以反推如果长度到8了，应该是key的分布被人介入了，也就是平时理解的拒绝服务攻击了。

# HashTable

HashTable 通过对 HashMap 的所有操作加 synchronized 锁来防止并发错误，类似于全表锁。

# ConcurrentHashMap

JDK1.7: ReentrantLock+Segment+HashEntry，分段锁。

- put: Segment实现了ReentrantLock,也就带有锁的功能，当执行put操作时，会进行第一次key的hash来定位Segment的位置，如果该Segment还没有初始化，即通过CAS操作进行赋值，然后进行第二次hash操作，找到相应的HashEntry的位置。
- get: get不加锁可以保证线程安全，使用了UNSAFE的getObjectVolatile具有读的volatile语义。

JDK1.8: synchronized+CAS+HashEntry+红黑树.

- 因为 synchronized 进行了升级，此时高并发下不亚于 ReentrantLock。
- 赋值时采用 CAS，遍历链表/红黑树时采用 synchronized。

ConcurrentHashMap 扩容

- put 对当前的table进行无条件自循环直到put成功。
  - 当检查当前hash桶在扩容时，会协助一起扩容。  else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);
  - 如果当前需要操作的节点还不是ForwardingNode即还没有完成扩容操作，那么会直接使用源tab。
- get
  - 会调用ForwardingNode的find方法，去nextTable里面查找相关元素。
