---
layout: post
title: ConcurrentHashMap源码解析——并发容器设计思路
date: 2018-11-07 21:08
comments: true
description: 读ConcurrentHashMap源码的一点收获
tags:
- java
- 读源码
---

>**001——背景**

&emsp;&emsp;读`ConcurrentHashMap`之前,先得了解一下`HashMap`的背景：JDK7的`HashMap`是用`table`+`链表`(拉链法)实现的,但JDK8之后发生了较大变化,采用`table`+`链表`+`红黑树`方式,让冲突的值能够在O(logn)的进行增删改查;<br>
&emsp;&emsp;另外`ConcurrentHashMap`是并发容器,内部重新进行了结构设计来保证线程安全,而不是像同步容器Collections.synchronizedMap()简单地对`HashMap`的方法加锁,因此`ConcurrentHashMap`的吞吐量是要高于Collections.synchronizedMap()的,接下来就来看看它是如何设计的吧：

<hr>

>**002——JDK7的ConcurrentHashMap源码分析**

### (1) 整体结构
&emsp;&emsp;整体的思路其实来源于`HashMap`的线程安全问题只存在于冲突的值,也就是相同hash值的那一个链表的值,因此对整体的操作加锁实际上是没有必要的,`ConcurrentHashMap`就是利用了这一点,把`HashMap`中的数组分割成多个`Segment`,然后对各个`Segment`进行加锁,保证了线程安全并且提高了吞吐量.<br>
&emsp;&emsp;`ConcurrentHashMap`的结构如图所示:(原图作者https://javadoop.com/post/hashmap)
![ConcurrentHashMap结构]({{ '/images/blog/3-1.png' | relative_url }})

### (2) Segment内部类
>Segment类
{:.filename}
{% highlight java linenos%}
    /**
    * Segment是特殊的Hash表,该类继承了ReentrantLock,用来简化锁操作
    */
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        ...
        //存储数据的链表数组,每个HashEntry都是一个链表头
        //volatile来保证线程可见性和顺序性
        transient volatile HashEntry<K,V>[] table;

        //Segment本质就是Hash表,拥有和HashMap几乎相同的操作
        final V put(K key, int hash, V value, boolean onlyIfAbsent) {...}
        final V remove(Object key, int hash, Object value) {...}
        final boolean replace(K key, int hash, V oldValue, V newValue) {...}
        final V replace(K key, int hash, V value) {...}
        final void clear() {...}

        //rehash操作,扩容+数据重新排列
        private void rehash(HashEntry<K,V> node) {...}

        //循环尝试加锁,后面会详细介绍这个方法
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {...}

        //和scanAndLockForPut类似的功能
        private void scanAndLock(Object key, int hash) {...}
        ...        
    }
{% endhighlight %}
&emsp;&emsp;可以看到`Segment`其实就是继承了`ReentrantLock`的一种特殊的Hash表,包含和`HashMap`几乎相同的操作,存储数据使用的是`HashEntry`,每个`HashEntry`包含一个键值对和一个指向下一个`HashEntry`的指针,其结构如下所示：
>HashEntry类
{:.filename}
{% highlight java linenos%}
    //HashEntry就是链表
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;   
        ...
    }
{% endhighlight %}
&emsp;&emsp;接下来看看`Segment`是如何保证线程安全性,首先是`put`操作:
>Segment.put()
{:.filename}
{% highlight java linenos%}
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        //尝试获取锁,并利用等待锁的时间创建节点node(节约new对象的时间),
        //这里的node有两种情况:
        //(1) tryLock()成功,此时无并发,所以不用提前new对象,往下走、
        //(2) tryLock()失败,存在并发,进入scanAndLockForPut重试多次,
        //    在等待并发的时候顺便new一个HashEntry给node
        //当然,上述需要new对象的场景仅发生在key不存在的情况,否则直接更新值就行了.
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;  //计算下标,要存放在哪个链表中
            HashEntry<K,V> first = entryAt(tab, index);  //获取当前链表头
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {  //当前链表存在
                    //这里很简单,就是找到对应的key节点,存入值
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
                else {  //如果找不到对应的key节点,就需要new一个HashEntry
                    if (node != null) //不为null说明在scanAndLockForPut中等待锁的空闲时间已经new好了对象
                        node.setNext(first); //头插法
                    else  //为null说明不存在并发,或者并发很快就结束了,没来得及new对象
                        node = new HashEntry<K,V>(hash, key, value, first); 
                    int c = count + 1;
                    //容量越限,需要扩容+重排列节点
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        setEntryAt(tab, index, node);
                    ++modCount;  //任何修改相关的操作后,都会让modCount++,用于并发下判断我的数据是否被别人动过
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();  //最终解锁
        }
        return oldValue;
    }
{% endhighlight %}

&emsp;&emsp;再来看看`put`一开始尝试获取锁的方法`scanAndLockForPut`都做了些什么:
>Segment.scanAndLockForPut()
{:.filename}
{% highlight java linenos%}
    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;  //node初始为null
        int retries = -1; // 重试次数
        //开始尝试加锁,如果此时并发消失,会直接退出,返回null
        while (!tryLock()) { 
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                //这里很简单,第一次重试失败后,就尝试创建对象
                //目的就是利用等待的时间来提前new好对象
                if (e == null) {
                    if (node == null) 
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            //如果重试多次都不行,就干脆lock来阻塞等待
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            //这里情况是在重试过程中,其他线程修改了该链表,那么就要重头再来
            else if ((retries & 1) == 0 &&
                        (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
{% endhighlight %}

&emsp;&emsp;`Segment`的`remove`,`replace`等操作思路和`put`类似,就不赘述了;奇怪的是`Segment`没有提供`get`操作,实际上`get`是在ConcurrentHashMap中直接实现的,接下来看看ConcurrentHashMap是如何设计的吧：

### (3) ConcurrentHashMap结构

&emsp;&emsp;先来看看ConcurrentHashMap构造器的参数有哪些:
> ConcurrentHashMap构造器
{:.filename}
{% highlight java linenos%}
public ConcurrentHashMap(int initialCapacity,
                        float loadFactor, 
                        int concurrencyLevel)
{% endhighlight %}

- initialCapacity 初始容量; 默认值16,这个很好理解,就是HashMap的整体容量,最后会平均分给各个Segment;
- loadFactor 负载因子; 默认值0.75,简单说就是超过75%的容量时会扩容;
- concurrencyLevel 并发量;默认值16,这个是ConcurrentHashMap独有的参数,实际上就是Segment是数量,因为每个Segment可以并发互不影响,因此其数量就是并发量.

&emsp;&emsp;在看构造器之前,需要先介绍一个概念,帮助理解构造器实现;到目前为止,我们知道了`ConcurrentHashMap`是包含多个`Segment`的,那么每次插入时是如何判断要操作哪个`Segment`的呢？<br>
&emsp;&emsp;`ConcurrentHashMap`使用了Key的hash值的`高n位`来确定操作哪个`Segment`的,比如说有16个`Segment`,那么需要log(16)=4个bit来进行`Segment`选择,因此hash值的高4位就用来进行段选,也就是说hash值需要右移32-4=28位来得到段选值,这一点很重要,理解之后就可以看看构造器的实现了：

> ConcurrentHashMap构造器
{:.filename}
{% highlight java linenos%}
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, 
                             int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_Segment)
            concurrencyLevel = MAX_Segment;
        // 这里假设 concurrencyLevel == 16
        int sshift = 0;  //段选需要多少位,这里就是log(concurrencyLevel) = 4 bit
        int ssize = 1;   //这个是Segment的个数
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.Segmenthift = 32 - sshift;  //这个值就是hash需要右移的位数了,32 - 4 = 28 bit
        this.segmentMask = ssize - 1;     /*这里是位屏蔽,也就是sshift个1,这里等于b'1111;
                                          如果是无符号右移的话,其实这个segmentMask没有什么作用,
                                          这里应该是怕hash值被有符号右移了吧,做一个与操作保证计算的
                                          段选值是正确的*/
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;  //这里是每个Segment的容量 = 总容量 / Segment个数
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        //这里就是初始化Segment数组,并且创建segment[0]
        //其他的Segment会在插入第一个键值对的时候创建
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of Segment[0]
        this.Segment = ss;
    }
{% endhighlight %}

&emsp;&emsp;然后看看ConcurrentHashMap的put操作:
> ConcurrentHashMap.put()
{:.filename}
{% highlight java linenos%}
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;  //Segment选择,hash无符号右移28位,然后和b'1111与操作
        //下面是UNSAFE包里面计算指针偏移的相关操作了,简单理解就是通过下标j来找到对应Segment
        if ((s = (Segment<K,V>)UNSAFE.getObject         
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        //最终找到调用Segment方法的put完成操作
        return s.put(key, hash, value, false);
    }
{% endhighlight %}

&emsp;&emsp;接下来是get操作:
> ConcurrentHashMap.get()
{:.filename}
{% highlight java linenos%}
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        //和put一样,先根据hash值找到对应的Segment
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        //由于rehash的存在,这里是通过UNSAFE包来获取volatile的Segment
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            //接下来通过hash值和tab.length与操作,找到对应的链表头,进行遍历
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
{% endhighlight %}

>**003——JDK8的ConcurrentHashMap结构**

&emsp;&emsp;JDK8采用了另外一种结构,不再使用分段锁,而是采用手动加锁的方式来确保线程安全;其实JDK7的缺点也很明显,就是我们需要自己指定`并发量`(即`Segment`的大小),很难去把控这个大小,而且一旦指定了就不能修改了;另外JDK8的`HashMap`和`ConcurrentHashMap`还有一点优化,就是之前提到的,当链表超过一定大小后会转换成红黑树,提升效率;以后有时间再分析一下JDK8的源码吧.

<hr>

>**004——总结**

- `Collections.synchronizedMap()`这种直接在方法上加`synchronized`的方式简单,但效率较低;相当于`synchronized(this)`或者`synchronized(class)`;
- JDK7的`ConcurrentHashMap`利用了Hash表在不同hash值之间的独立性,使用分段锁提高的并发量;
- 设计并发类时,多用final,能提高效率;对于有些公共访问的域,记得用`volatile`保证可见性和有序性(但不保证原子性);
- new 对象一般是比较耗时的,等待lock的时间我们可以把需要new的对象提前new好(借鉴了`scanAndLockForPut()`)
- 在遍历且没有加锁时,很有可能被别的线程修改结构,因此可以用modCount累加来判断是否发生了修改.

