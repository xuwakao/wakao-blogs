### 官方文档总结
>This class implements a hash table, which maps keys to values. __Any non-null object can be used as a key or as a value.__
>
>To successfully store and retrieve objects from a hashtable, the objects used as keys must implement the hashCode method and the equals method.
>
>An instance of Hashtable has two parameters that affect its performance: initial capacity and load factor. The capacity is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. Note that the hash table is open: in the case of a "hash collision", a single bucket stores multiple entries, which must be searched sequentially. The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased. The initial capacity and load factor parameters are merely hints to the implementation. The exact details as to when and whether the rehash method is invoked are implementation-dependent.
>
>Generally, the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the time cost to look up an entry (which is reflected in most Hashtable operations, including get and put).
>
>The initial capacity controls a tradeoff between wasted space and the need for rehash operations, which are time-consuming. No rehash operations will ever occur if the initial capacity is greater than the maximum number of entries the Hashtable will contain divided by its load factor. However, setting the initial capacity too high can waste space.
>
>If many entries are to be made into a Hashtable, creating it with a sufficiently large capacity may allow the entries to be inserted more efficiently than letting it perform automatic rehashing as needed to grow the table.
>
>This example creates a hashtable of numbers. It uses the names of the numbers as keys:
>   ```Hashtable<String, Integer> numbers
     = new Hashtable<String, Integer>();
   numbers.put("one", 1);
   numbers.put("two", 2);
   numbers.put("three", 3);```
>
>To retrieve a number, use the following code:
>
>   ```Integer n = numbers.get("two");
   if (n != null) {
     System.out.println("two = " + n);
   }```
>The iterators returned by the iterator method of the collections returned by all of this class's "collection view methods" are fail-fast: if the Hashtable is structurally modified at any time after the iterator is created, in any way except through the iterator's own remove method, the iterator will throw a ConcurrentModificationException. Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future. The Enumerations returned by Hashtable's keys and elements methods are not fail-fast.
>
>Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.
>
>As of the Java 2 platform v1.2, this class was retrofitted to implement the Map interface, making it a member of the Java Collections Framework. Unlike the new collection implementations, __Hashtable is synchronized. If a thread-safe implementation is not needed, it is recommended to use HashMap in place of Hashtable. If a thread-safe highly-concurrent implementation is desired, then it is recommended to use ConcurrentHashMap in place of Hashtable.__

### 特点

![继承关系](https://images0.cnblogs.com/blog/497634/201401/280030194375782.jpg)

从官方文档说明，知道HashMap有以下特点：
* 1 不允许任何NULL key和value
* 2 线程安全，每一个操作方法都用sychronized修饰（高并发情况下，建议使用ConcurrentHashMap）


### 数据结构

<img src="https://images2018.cnblogs.com/blog/137084/201804/137084-20180422151410659-560763443.jpg" height="300" width="600">

### 源码说明

* 1 capacity就是bucket数量（默认大小11）
* 2 threshold即元素数量阈值，超过就扩容（等于capacity* load factor）
* 3 扩容的方式是2倍再加1


### 源码分析

* #### hash算法
```
int index = (hash & 0x7FFFFFFF) % tab.length;
```
hash算法很简单，就是数组长度求模，与0x7FFFFFFF的作用是，去除最高位的sign位，防止出现负数；

* ####rehash函数

```
    /**
     * Increases the capacity of and internally reorganizes this
     * hashtable, in order to accommodate and access its entries more
     * efficiently. This method is called automatically when the
     * number of keys in the hashtable exceeds this hashtable's capacity
     * and load factor.
     */
    @SuppressWarnings("unchecked")
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;
        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1; //扩容方式是乘以2，再加1
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];
        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;
        for (int i = oldCapacity ; i-- > 0 ;) {//重新rehash所有节点，性能低下的原因
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;
                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```