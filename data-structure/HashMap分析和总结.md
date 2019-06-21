###官方文档总结

> 
Hash table based implementation of the Map interface. This implementation provides all of the optional map operations, and ***permits null values and the null key***. (The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls.) ***This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time***.
This implementation provides constant-time performance for the basic operations (get and put), assuming the hash function disperses the elements properly among the buckets. Iteration over collection views requires time proportional to the "capacity" of theHashMap instance (the number of buckets) plus its size (the number of key-value mappings). Thus, it's very important not to set the initial capacity too high (or the load factor too low) if iteration performance is important.
An instance of HashMap has two parameters that affect its performance: initial capacity and load factor. The capacity is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. The load factoris a measure of how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.
As a general rule, the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the HashMap class, including get and put). The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations. If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.
If many mappings are to be stored in a HashMap instance, creating it with a sufficiently large capacity will allow the mappings to be stored more efficiently than letting it perform automatic rehashing as needed to grow the table. Note that using many keys with the same hashCode() is a sure way to slow down performance of any hash table. To ameliorate impact, when keys are Comparable, this class may use comparison order among keys to help break ties.
***Note that this implementation is not synchronized.*** If multiple threads access a hash map concurrently, and at least one of the threads modifies the map structurally, it must be synchronized externally. (A structural modification is any operation that adds or deletes one or more mappings; merely changing the value associated with a key that an instance already contains is not a structural modification.) This is typically accomplished by synchronizing on some object that naturally encapsulates the map. If no such object exists, the map should be "wrapped" using the Collections.synchronizedMap method. This is best done at creation time, to prevent accidental unsynchronized access to the map:
   `Map m = Collections.synchronizedMap(new HashMap(...));`
The iterators returned by all of this class's "collection view methods" are fail-fast: if the map is structurally modified at any time after the iterator is created, in any way except through the iterator's own remove method, the iterator will throw aConcurrentModificationException. Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future.
Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationExceptionon a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.
This class is a member of the Java Collections Framework.
Pasted from: https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html#HashMap--

###特点

![继承关系](https://static.javatpoint.com/images/hashmap.png)

从官方文档说明，知道``HashMap``有以下特点：
* 1 允许NULL key和value（这是和``HashTable``的区别之一，``HashTable``实现基本等价``HashMap``））
* 2 非线程安全（这是和``HashTable``的区别之一）
* 3 无序列表（插入顺序和遍历的顺序是不一致的）

###数据结构

<img src="http://www.programmersought.com/images/232/f352701e5bd28a4a7f58d4b58aa4eff8.png" height="600 " width="1000" title="JDK 1.7前的结构">

<img src="http://www.programmersought.com/images/38/243d588fc25bcd1462ad067dd312cf16.png" height="600 " width="1000" title="JDK 1.8后的结构">

###源码说明

* 1 ``capacity``就是``table``数组容量（默认大小16），必须是2的n次方（源码中的``table``，是一个``node``数组）
* 2 ``threshold``即元素数量阈值，超过就扩容（等于``capacity* load factor``）
* 3 JDK1.8后，``bucket``内部根据节点数量不同，结构不同，8或以下采用链表，8以上采用红黑树（主要是，链表查找效率是$O(N)$，在N较大的时候效率低下，红黑色是$O(lgN)$

###源码分析

* ####hash算法
```
 /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
上面函数中说明，在``HashMap``的设计中，__``tableSize``的大小必定是2的N次方__。
这样的好处是，$(2^{n}-1)$刚好是一个低位掩码（2进制全是1），当查找``table``数组索引时，需要进行求模运算，可以对``hash``和$(2^{n}-1)$进行与操作代替掉求模运算（位运算比算术运算高效）。
``` 源码中的求模运算
tab[i = (n - 1) & hash]
```
上面的求模使用位运算，因为是进行与操作，而且hash值与对象是一个类似掩码的数，那么运算结果会使hash的高位全部置零，只保留了低几位，这样会导致一个严重问题就是，如果hash值是不那么理想的hash函数产生，这很容易出现散列表的碰撞问题，导致效率低下。

<img src="https://images2017.cnblogs.com/blog/1149791/201708/1149791-20170811095824214-1137254686.png" height="400 " width="600">

为了防止该问题，``HashMap``使用一个扰动函数处理该问题：
```Hash扰动函数
/**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower. Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.) So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

该扰动函数就是，对原``hash``右移16位（int值hash是32位），然后用该值和原``hash``进行异或运算，作用的让高位和低位充分混合，加大随机性，也使高位信息参与``hash``过程。参见下图：
<img src="https://images2017.cnblogs.com/blog/1149791/201708/1149791-20170811095904120-1554448399.png" height="400 " width="600" title="扰动函数">


* ####resize函数

```resize函数
    /**
     * Initializes or doubles table size. If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold     使用提供的threshold初始化，cap=threshold
            newCap = oldThr;
        else { // zero initial threshold signifies using defaults     使用默认值初始化，cap=16，factor=0.75f
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) { //复制旧的元素到新table
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {  // hash在oldCap-1高一位bit为0的情况，对应 newIndex=oldIndex，不移动位置（详解看下面）
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {   // hash在oldCap-1高一位bit为1的情况，对应 newIndex= oldIndex + oldCap，不移动位置
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
``resize``函数主要是初始化``table``数组，或者需要扩容时，替换旧``table``并复制旧元素到新``table``上。

``resize``是一个耗性能的操作，因为``node``节点需要``rehash``，计算过程如何：

__注意 ：mod的分配率公式：$d \pmod{a*b*c} = (d \pmod{a}) + a[gcd(d, a) \pmod{b}] + ab[gcd(gcd(d,a),b) \pmod{c}]$ (\是欧几里得除法)__

$old index = hash \pmod{2^{(n)}} $
$new index = hash \pmod{2^{(n+1)}} $
$new index = hash \pmod{2^{(n)}} + 2^{(n)}[gcd(hash, n) \pmod{2}] + 2^{(n+1)}[gcd[gcd(hash, 2^{(n)}], 2) \pmod{1}]$
由于任意数mod 2，非0即1，mod 1，肯定是0，所以：

$new index = old index $或者$new index = old index + 2^{(n)}$

$2^{(n)}$ 其实就是旧``table``长度

__结论：__

$new index = old index$  或者  $new index = old index + (old capacity)$

在二进制上，借用美图的一张图，这过程可以简化，如下：
![](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/4d8022db.png)

___由此可见，new index是和hash在oldCap-1高一位bit的值相关。___

___rehash的整个过程如下图：___
![](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/3cc9813a.png)

___注意：不要被图误导有奇偶性，这个rehash过程，移动的位置根据的是``hash``的那个bit位，而这个bit位是随机性的。___

* ####put函数
```putval函数
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null); //不碰撞，直接新建node
        else { //碰撞情况
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p; //添加记录和头结点一致，后续看是否更换该节点的value值
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); //如果是红黑节点，则在红黑色内进行put操作
            else {
                for (int binCount = 0; ; ++binCount) { //在链表或者红黑色中查找节点，找到就后续替换，找不到就添加到链表或者红黑色
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
            if (e != null) { // existing mapping for key   根据onlyIfAbsent控制，是否替换旧节点的value值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold) //添加新节点后，判断是否需要扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```