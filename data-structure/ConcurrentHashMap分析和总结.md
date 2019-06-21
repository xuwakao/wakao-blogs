### 官方文档总结

>A hash table supporting full concurrency of retrievals and high expected concurrency for updates. This class obeys the same functional specification as Hashtable, and includes versions of methods corresponding to each method of Hashtable. However, __even though all operations are thread-safe, retrieval operations do not entail locking, and there is not any support for locking the entire table in a way that prevents all access. This class is fully interoperable with Hashtable in programs that rely on its thread safety but not on its synchronization details.__
>
>Retrieval operations (including get) generally do not block, so may overlap with update operations (including put and remove). Retrievals reflect the results of the most recently completed update operations holding upon their onset. (More formally, an update operation for a given key bears a happens-before relation with any (non-null) retrieval for that key reporting the updated value.) For aggregate operations such as putAll and clear, concurrent retrievals may reflect insertion or removal of only some entries. Similarly, Iterators, Spliterators and Enumerations return elements reflecting the state of the hash table at some point at or since the creation of the iterator/enumeration. They do not throw ConcurrentModificationException. However, iterators are designed to be used by only one thread at a time. Bear in mind that the results of aggregate status methods including size, isEmpty, and containsValue are typically useful only when a map is not undergoing concurrent updates in other threads. Otherwise the results of these methods reflect transient states that may be adequate for monitoring or estimation purposes, but not for program control.
>
>The table is dynamically expanded when there are too many collisions (i.e., keys that have distinct hash codes but fall into the same slot modulo the table size), with the expected average effect of maintaining roughly two bins per mapping (corresponding to a 0.75 load factor threshold for resizing). There may be much variance around this average as mappings are added and removed, but overall, this maintains a commonly accepted time/space tradeoff for hash tables. However, resizing this or any other kind of hash table may be a relatively slow operation. When possible, it is a good idea to provide a size estimate as an optional initialCapacity constructor argument. An additional optional loadFactor constructor argument provides a further means of customizing initial table capacity by specifying the table density to be used in calculating the amount of space to allocate for the given number of elements. Also, for compatibility with previous versions of this class, constructors may optionally specify an expected concurrencyLevel as an additional hint for internal sizing. Note that using many keys with exactly the same hashCode() is a sure way to slow down performance of any hash table. To ameliorate impact, when keys are Comparable, this class may use comparison order among keys to help break ties.
>
>A Set projection of a ConcurrentHashMap may be created (using newKeySet() or newKeySet(int)), or viewed (using keySet(Object) when only keys are of interest, and the mapped values are (perhaps transiently) not used or all take the same mapping value.
>
>A ConcurrentHashMap can be used as scalable frequency map (a form of histogram or multiset) by using LongAdder values and initializing via computeIfAbsent. For example, to add a count to a ConcurrentHashMap<String,LongAdder> freqs, you can use freqs.computeIfAbsent(k -> new LongAdder()).increment();
>
>This class and its views and iterators implement all of the optional methods of the Map and Iterator interfaces.
>
>___Like Hashtable but unlike HashMap, this class does not allow null to be used as a key or value.___
>
>ConcurrentHashMaps support a set of sequential and parallel bulk operations that, unlike most Stream methods, are designed to be safely, and often sensibly, applied even with maps that are being concurrently updated by other threads; for example, when computing a snapshot summary of the values in a shared registry. There are three kinds of operation, each with four forms, accepting functions with Keys, Values, Entries, and (Key, Value) arguments and/or return values. ___Because the elements of a ConcurrentHashMap are not ordered in any particular way, and may be processed in different orders in different parallel executions, the correctness of supplied functions should not depend on any ordering, or on any other objects or values that may transiently change while computation is in progress; and except for forEach actions, should ideally be side-effect-free.___ Bulk operations on Map.Entry objects do not support method setValue.
>
>* forEach: Perform a given action on each element. A variant form applies a given transformation on each element before performing the action.
>* search: Return the first available non-null result of applying a given function on each element; skipping further search when a result is found.
>* reduce: Accumulate each element. The supplied reduction function cannot rely on ordering (more formally, it should be both associative and commutative). There are five variants:
>   * Plain reductions. (There is not a form of this method for (key, value) function arguments since there is no corresponding return type.)
>   * Mapped reductions that accumulate the results of a given function applied to each element.
>   * Reductions to scalar doubles, longs, and ints, using a given basis value.
>
>These bulk operations accept a parallelismThreshold argument. Methods proceed sequentially if the current map size is estimated to be less than the given threshold. Using a value of Long.MAX_VALUE suppresses all parallelism. Using a value of 1 results in maximal parallelism by partitioning into enough subtasks to fully utilize the ForkJoinPool.commonPool() that is used for all parallel computations. Normally, you would initially choose one of these extreme values, and then measure performance of using in-between values that trade off overhead versus throughput.
>
>The concurrency properties of bulk operations follow from those of ConcurrentHashMap: Any non-null result returned from get(key) and related access methods bears a happens-before relation with the associated insertion or update. The result of any bulk operation reflects the composition of these per-element relations (but is not necessarily atomic with respect to the map as a whole unless it is somehow known to be quiescent). Conversely, because keys and values in the map are never null, null serves as a reliable atomic indicator of the current lack of any result. To maintain this property, null serves as an implicit basis for all non-scalar reduction operations. For the double, long, and int versions, the basis should be one that, when combined with any other value, returns that other value (more formally, it should be the identity element for the reduction). Most common reductions have these properties; for example, computing a sum with basis 0 or a minimum with basis MAX_VALUE.
>
>Search and transformation functions provided as arguments should similarly return null to indicate the lack of any result (in which case it is not used). In the case of mapped reductions, this also enables transformations to serve as filters, returning null (or, in the case of primitive specializations, the identity basis) if the element should not be combined. You can create compound transformations and filterings by composing them yourself under this "null means there is nothing there now" rule before using them in search or reduce operations.
>
>Methods accepting and/or returning Entry arguments maintain key-value associations. They may be useful for example when finding the key for the greatest value. Note that "plain" Entry arguments can be supplied using new AbstractMap.SimpleEntry(k,v).
Bulk operations may complete abruptly, throwing an exception encountered in the application of a supplied function. Bear in mind when handling such exceptions that other concurrently executing functions could also have thrown exceptions, or would have done so if the first exception had not occurred.
>
>Speedups for parallel compared to sequential forms are common but not guaranteed. Parallel operations involving brief functions on small maps may execute more slowly than sequential forms if the underlying work to parallelize the computation is more expensive than the computation itself. Similarly, parallelization may not lead to much actual parallelism if all processors are busy performing unrelated tasks.
>
>All arguments to all task methods must be non-null.
>
>This class is a member of the Java Collections Framework.

### 特点

继承关系

![](https://user-gold-cdn.xitu.io/2017/12/13/1605097186c4c5a4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


从官方文档说明，知道``HashMap``有以下特点：
* 1 不允许NULL key和value（这是和``HashTable``的一致）
* 2 线程安全，但是无锁
* 3 无序列表（插入顺序和遍历的顺序是不一致的）
* 4 支持批量操作（bulk operation）

### 数据结构

<img src="http://www.programmersought.com/images/232/f7a1ec4d9fc9de85ef87973d4d4cfb10.png" height="600 " width="1000" title="JDK 1.7前的结构">

<img src="http://www.programmersought.com/images/38/243d588fc25bcd1462ad067dd312cf16.png" height="600 " width="1000" title="JDK 1.8后的结构">

### 源码说明

* 1 ``capacity``就是``table``数组大小（默认大小16），可以根据传递参数根据规律调整，必须是$2^{(n)}$（源码中的``table``，是一个``node``数组）
* 2 ``sizeCtl``，该变量根据大小不同，有多重含义（下面详解）；
* 3 JDK1.8后，bucket内部根据节点数量不同，结构不同，8或以下采用链表，8以上采用红黑树（主要是，链表查找效率是$O(N)$，在N较大的时候效率低下，红黑色是$O(lgN)$
* 4 ``ForwardingNode``扩容时，替换被迁移节点，用于标识当前节点正在迁移中；

### 源码分析


* ##### 初始容量

有两个构造函数可以指定table数组的初始容量：
```
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel) // Use at least as many bins
            initialCapacity = concurrencyLevel; // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    } 
```
奇怪的是，计算``capacity``的方法竟然是不一样的，而且传递的参数往往不是实际``table``的大小，需要经过``tableSizeFor``函数转换。
两种方式分别是
``` 
long size = (long)(1.0 + (long)initialCapacity / loadFactor);
```
和
```
tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1)); 
```
这里其实是一个optimization trick。
* 1 第一个方法，假如``loadFactor=0.75``，那么size为2.33倍``initialCapacity``；
* 2 第二个方法，``(initialCapacity >>> 1)``其实就是除以2（利用位运算的高效），那么size为2.5倍``initialCapacity``；+1目的是补偿右移操作早上类似于乘以2.5f的的取整损失；

初始容量需要经过``tableSizeFor``函数转换：
```
    /**
     * Returns a power of two table size for the given desired capacity.
     * See Hackers Delight, sec 3.2
     */
    private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    } 
```
和HashMap的处理一样，就是取最近的$2^{(n)}$的数。



* ##### sizeCtl和threshold

* 1 ``table``未初始化时：$sizeCtl = 0$ 或者 $sizeCtl= capacity$;
* 2 ``table``正在初始化：$sizeCtl = -1$;
* 3 ``table``初始化完成：$sizeCtl = thresold$;
* 4 当$sizeCtl = -(1 + N) $，表明正在有N条线程正在进行``resize``操作；(___这是注释的说明，但是这个说明很含糊而且不太准确，后面解释___)

有意思的一个地方是，``sizeCtl``计算经常是如下代码：
```
sizeCtl = n - (n >>> 2);
```
其实就是 $sizeCtl =thresold = capacity * 0.75 $ 的意思，0.75等价于``loadFactor``；

在注释中，说：
```
/**
     * Table initialization and resizing control. When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads). Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     */
```
在 $sizeCtl < -1 $ 的情况下，注释中给人的错觉就像是：$\left|sizeCtl\right| - 1$ 就是正在``resize``的线程数，实际上不然。

实际上，``sizeCtl`` 分成高低位，高低位分解由下面的``RESIZE_STAMP_BITS ``变量定义：

```
    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;
    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
```
___sizeCtl分解___
高16位|低16位
-|-
正在扩容的标识（负值）|并发扩容的线程数


* ##### hash算法

``ConcurrentHashMap``的hash算法如下：
```
 /**
     * Spreads (XORs) higher bits of hash to lower and also forces top
     * bit to 0. Because the table uses power-of-two masking, sets of
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
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

这里的算法和``HashMap``原理上一样，但是多了``HASH_BITS``的与运算：
```
 /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED = -1; // hash for forwarding nodes
    static final int TREEBIN = -2; // hash for roots of trees
    static final int RESERVED = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```
其实，``HASH_BITS``的作用是去取最高的符号位，保证必须为整数，主要原因是（个人猜测），防止和``MOVED ``，``TREEBIN ``，``RESERVED``这些预留hard-code的``hash``为负数的节点产生碰撞。


* ##### initTable初始化

```
    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin 多线程同时初始化（即sizeCtl = -1 )时，竞争失败的线程一直循环yield
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  //CAS设置 sizeCtl = -1 
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];  //初始化table大小为DEFAULT_CAPACITY或者sizeCtl
                        table = tab = nt;
                        sc = n - (n >>> 2); // 计算threshold
                    }
                } finally {
                    sizeCtl = sc; //设置sizeCtl 为threshold
                }
                break;
            }
        }
        return tab;
    } 
```

* ##### put函数

```
    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p>The value can be retrieved by calling the {@code get} method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with {@code key}, or
     * {@code null} if there was no mapping for {@code key}
     * @throws NullPointerException if the specified key or value is null
     */
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
            if (tab == null || (n = tab.length) == 0)  //初始化table
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  //当前节点为NULL，CAS设置新头节点
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break; // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED) //正在resize过程（resize时，头结点的hash未MOVED值）
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) { //对当前节点加锁
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { //normal节点（非红黑树，非MOVED，非Reserved节点）
                            binCount = 1; //头结点占了一个节点，所以初始为1，该值用于计算bin内节点个数
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) { //判断是否存在重复的旧节点，并看onlyIfAbsent是否替换
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) { //在尾部添加节点
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { //节点为红黑树节点，通过红黑树添加节点
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD) //判断bin内节点是否多于TREEIFY_THRESHOLD=8，超过就要看是否转换红黑树，是否转要看treeifyBin
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount); //增加节点数量记录
        return null;
    }
```

* ##### treeifyBin函数

``treeifyBin``函数对于节点数量过多的处理，是有别于HashMap的。
* 1.``table``比较小时（小于``MIN_TREEIFY_CAPACITY=64``），而是先``resize``；
* 2.``table``达到一定数量时，转换红黑树；

```
    /**
     * Replaces all linked nodes in bin at given index unless table is
     * too small, in which case resizes instead.
     */
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1); // 如果table还比较小（MIN_TREEIFY_CAPACITY=64），先不转换红黑树，先resize
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) { //转换成红黑树
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

* ##### tryPresize函数
```
    /**
     * Tries to presize table to accommodate the given number of elements.
     *
     * @param size number of elements (doesn't need to be perfectly accurate)
     */
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1); //扩大容量
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) { //未初始化table的情况
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // -1即未初始化table时对应的sizeCtl
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; //新建table
                            table = nt;
                            sc = n - (n >>> 2); //thresold的计算
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) { //没有并发的其他操作，导致table已经变化了
                int rs = resizeStamp(n); //这是一个神奇的函数，下面讲解
                if (sc < 0) { //这代码真的难懂，以为自己脑子不够用，实际上这个分支根本不会执行，前面sc(local variable不存在竞态问题)就已经是保证大于等于0，issue:https://bugs.openjdk.java.net/browse/JDK-8215409,https://bugs.openjdk.java.net/browse/JDK-8203786
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2)) //这个神操作暂时看不懂，只知道如果设置成功，就会是一个负数，为什么+2也搞不懂
                    transfer(tab, null); // 真正的复制节点操作
            }
        }
    }
```

* ##### resizeStamp函数
```
    /**
     * Returns the stamp bits for resizing a table of size n.
     * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
     */
    static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```
``Integer.numberOfLeadingZeros(n)``获取高位的0个数，由于int值就32位，那么这个结果必定少于32。（这里n参数其实都是``table``长度，间接记录了``table``的长度）。
然后按位与操作，另一位操作数是``0000 0000 0000 0000 1000 0000 0000 0000``，
那么返回结果，肯定是一个格式为 ：``0000 0000 0000 0000 1000 0000 00xx xxxx`` 的int值。

``resizeStamp``返回结果，一般被调用方，进行一下操作：
```
resizeStamp(n) << RESIZE_STAMP_SHIFT
```
那么必然会得到一个``1000 0000 00xx xxxx 0000 0000 0000 0000``，``sign bit``固定1，所以必定是一个负数，作用就是产生一个用于标识正在扩容的``sizeCtl``值（一个负数）。

``0000 0000 0000 0000 1000 0000 00xx xxxx``这个格式的int值，低六位被可能被占用的，这恰好对应说明了的这段注释，``RESIZE_STAMP_BITS``必须至少是6：

```
    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;
```



* ##### transfer函数，即扩容过程

首节点的黄色节点的第X位（X的值下面源码会说明）为0，蓝色节点第X为1。
``table``情况如下：
![](https://github.com/xuwakao/wakao-assets/blob/master/concurrenthashmap-table.jpg?raw=true)
* 首节点的黄色节点的第X位（X的值下面源码会说明）为0，蓝色节点第X为1。


``index=10``的节点，节点个数大于8，可能会触发``transfer``（可以参看前面``treeifyBin``），节点``transfer``执行如下：

![](https://github.com/xuwakao/wakao-assets/blob/master/concurrenthashmap-transfer-node.jpg?raw=true)

链表迁移情况如下：
![](https://github.com/xuwakao/wakao-assets/blob/master/concurrenthashmap-transfer-after.jpg?raw=true)


源码解释如下：

```
    /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) { // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) { // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;//倒序table处理扩容
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) { 
                int sc;
                if (finishing) { //所有线程transfer完后，更新table
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null) //空节点，设置为forward节点，不处理
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED) //该节点正在迁移
                advance = true; // already processed
            else {
                synchronized (f) { //节点内部链表的rehash过程，具体看上面讲解图
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n; //首节点的某个bit位X（其实就是2^n的n+1的bit位，X=n+1）的值，根据该值，下面将当前链表拆分成两条链表
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n; 
                                if (b != runBit) { //找出临近节点的X bit位不同的最后一个节点（这里实例即runBit=1，lastRun=5节点）
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) { //红黑色节点的处理情况
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```