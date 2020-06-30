### ConcurrentHashMap实现原理

　　众所周知，哈希表是中非常高效，复杂度为O(1)的数据结构，在Java开发中，我们最常见到最频繁使用的就是HashMap和HashTable，但是在线程竞争激烈的并发场景中使用都不够合理。

　　**HashMap** ：先说HashMap，HashMap是**线程不安全**的，在并发环境下，可能会形成**环状链表**（扩容时可能造成，具体原因自行百度google或查看源码分析），导致get操作时，cpu空转，所以，在并发环境中使用HashMap是非常危险的。

　　**HashTable** ： HashTable和HashMap的实现原理几乎一样，差别无非是**1.HashTable不允许key和value为null；2.HashTable是线程安全的。**但是HashTable线程安全的策略实现代价却太大了，简单粗暴，get/put所有相关操作都是synchronized的，这相当于给整个哈希表加了一把**大锁**，多线程访问时候，只要有一个线程访问或操作该对象，那其他线程只能阻塞，相当于将所有的操作**串行化**，在竞争激烈的并发场景中性能就会非常差。

![img](https://images2015.cnblogs.com/blog/1024555/201705/1024555-20170514173954488-1353945142.png)

　　HashTable性能差主要是由于所有操作需要竞争同一把锁，而如果容器中有多把锁，每一把锁锁一段数据，这样在多线程访问时不同段的数据时，就不会存在锁竞争了，这样便可以有效地提高并发效率。这就是ConcurrentHashMap所采用的"**分段锁**"思想。

　　![img](https://images2015.cnblogs.com/blog/1024555/201705/1024555-20170514174100832-1891630860.png)

### ConcurrentHashMap源码分析　　 

ConcurrentHashMap采用了非常精妙的"分段锁"策略，ConcurrentHashMap的主干是个Segment数组。

```java
 final Segment<K,V>[] segments;
```

　　Segment继承了ReentrantLock，所以它就是一种可重入锁（ReentrantLock)。在ConcurrentHashMap，一个Segment就是一个子哈希表，Segment里维护了一个HashEntry数组，并发环境下，对于不同Segment的数据进行操作是不用考虑锁竞争的。（就按默认的ConcurrentLeve为16来讲，理论上就允许16个线程并发执行，有木有很酷）

　　**所以，对于同一个Segment的操作才需考虑线程同步，不同的Segment则无需考虑。**

Segment类似于HashMap，一个Segment维护着一个HashEntry数组

```java
 transient volatile HashEntry<K,V>[] table;
```

HashEntry是目前我们提到的最小的逻辑处理单元了。一个ConcurrentHashMap维护一个Segment数组，一个Segment维护一个HashEntry数组。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
 static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        //其他省略
}    
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

我们说Segment类似哈希表，那么一些属性就跟我们之前提到的HashMap差不离，比如负载因子loadFactor，比如阈值threshold等等，看下Segment的构造方法

```java
Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
            this.loadFactor = lf;//负载因子
            this.threshold = threshold;//阈值
            this.table = tab;//主干数组即HashEntry数组
        }
```

我们来看下ConcurrentHashMap的构造方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
 1  public ConcurrentHashMap(int initialCapacity,
 2                                float loadFactor, int concurrencyLevel) {
 3           if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
 4               throw new IllegalArgumentException();
 5           //MAX_SEGMENTS 为1<<16=65536，也就是最大并发数为65536
 6           if (concurrencyLevel > MAX_SEGMENTS)
 7               concurrencyLevel = MAX_SEGMENTS;
 8           //2的sshif次方等于ssize，例:ssize=16,sshift=4;ssize=32,sshif=5
 9          int sshift = 0;
10          //ssize 为segments数组长度，根据concurrentLevel计算得出
11          int ssize = 1;
12          while (ssize < concurrencyLevel) {
13              ++sshift;
14              ssize <<= 1;
15          }
16          //segmentShift和segmentMask这两个变量在定位segment时会用到，后面会详细讲
17          this.segmentShift = 32 - sshift;
18          this.segmentMask = ssize - 1;
19          if (initialCapacity > MAXIMUM_CAPACITY)
20              initialCapacity = MAXIMUM_CAPACITY;
21          //计算cap的大小，即Segment中HashEntry的数组长度，cap也一定为2的n次方.
22          int c = initialCapacity / ssize;
23          if (c * ssize < initialCapacity)
24              ++c;
25          int cap = MIN_SEGMENT_TABLE_CAPACITY;
26          while (cap < c)
27              cap <<= 1;
28          //创建segments数组并初始化第一个Segment，其余的Segment延迟初始化
29          Segment<K,V> s0 =
30              new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
31                               (HashEntry<K,V>[])new HashEntry[cap]);
32          Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
33          UNSAFE.putOrderedObject(ss, SBASE, s0); 
34          this.segments = ss;
35      }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　初始化方法有三个参数，如果用户不指定则会使用默认值，initialCapacity为16，loadFactor为0.75（负载因子，扩容时需要参考），concurrentLevel为16。

　　**从上面的代码可以看出来,Segment数组的大小ssize是由concurrentLevel来决定的，但是却不一定等于concurrentLevel，ssize一定是大于或等于concurrentLevel的最小的2的次幂。比如：默认情况下concurrentLevel是16，则ssize为16；若concurrentLevel为14，ssize为16；若concurrentLevel为17，则ssize为32。为什么Segment的数组大小一定是2的次幂？其实主要是便于通过按位与的散列算法来定位Segment的index。至于更详细的原因，有兴趣的话可以参考我的另一篇文章《[HashMap实现原理及源码分析](http://www.cnblogs.com/chengxiao/p/6059914.html)》，其中对于数组长度为什么一定要是2的次幂有较为详细的分析。**

　　接下来，我们来看看put方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
 public V put(K key, V value) {
        Segment<K,V> s;
        //concurrentHashMap不允许key/value为空
        if (value == null)
            throw new NullPointerException();
        //hash函数对key的hashCode重新散列，避免差劲的不合理的hashcode，保证散列均匀
        int hash = hash(key);
        //返回的hash值无符号右移segmentShift位与段掩码进行位运算，定位segment
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　从源码看出，put的主要逻辑也就两步：**1.定位segment并确保定位的Segment已初始化 2.调用Segment的put方法。**

　**关于segmentShift和segmentMask**

　　segmentShift和segmentMask这两个全局变量的主要作用是用来定位Segment，int j =(hash >>> segmentShift) & segmentMask。

　　**segmentMask**：段掩码，假如segments数组长度为16，则段掩码为16-1=15；segments长度为32，段掩码为32-1=31。这样得到的所有bit位都为1，可以更好地保证散列的均匀性

　　**segmentShift**：2的sshift次方等于ssize，segmentShift=32-sshift。若segments长度为16，segmentShift=32-4=28;若segments长度为32，segmentShift=32-5=27。而计算得出的hash值最大为32位，无符号右移segmentShift，则意味着只保留高几位（其余位是没用的），然后与段掩码segmentMask位运算来定位Segment。

　　**get/put方法**

　　get方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
 public V get(Object key) {
        Segment<K,V> s; 
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        //先定位Segment，再定位HashEntry
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　**get方法无需加锁，由于其中涉及到的共享变量都使用volatile修饰，volatile可以保证内存可见性，所以不会读取到过期数据。**

　　来看下concurrentHashMap代理到Segment上的put方法，Segment中的put方法是要加锁的。只不过是锁粒度细了而已。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);//tryLock不成功时会遍历定位到的HashEnry位置的链表（遍历主要是为了使CPU缓存链表），若找不到，则创建HashEntry。tryLock一定次数后（MAX_SCAN_RETRIES变量决定），则lock。若遍历过程中，由于其他线程的操作导致链表头结点变化，则需要重新遍历。
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;//定位HashEntry，可以看到，这个hash值在定位Segment时和在Segment中定位HashEntry都会用到，只不过定位Segment时只用到高几位。
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
　　　　　　　　　　　　　　//若c超出阈值threshold，需要扩容并rehash。扩容后的容量是当前容量的2倍。这样可以最大程度避免之前散列好的entry重新散列，具体在另一篇文章中有详细分析，不赘述。扩容并rehash的这个过程是比较消耗资源的。
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#  总结

　　ConcurrentHashMap作为一种线程安全且高效的哈希表的解决方案，尤其其中的"分段锁"的方案，相比HashTable的全表锁在性能上的提升非常之大。本文对ConcurrentHashMap的实现原理进行了详细分析，并解读了部分源码，希望能帮助到有需要的童鞋。