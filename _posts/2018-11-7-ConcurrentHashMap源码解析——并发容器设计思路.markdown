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

&emsp;&emsp;读`ConcurrentHashMap`之前，先得了解一下`HashMap`的背景：JDK7的`HashMap`是用`table`+`链表`(拉链法)实现的，但JDK8之后发生了较大变化，采用`table`+`红黑树`方式，让冲突的值能够在O(logn)的进行增删改查；<br>
&emsp;&emsp;另外`ConcurrentHashMap`是并发容器，内部重新进行了结构设计来保证线程安全，而不是像同步容器Collections.synchronizedMap()简单地对`HashMap`的方法加锁，因此`ConcurrentHashMap`的吞吐量是要高于Collections.synchronizedMap()的，接下来就来看看它是如何设计的吧：

>**002——JDK7的ConcurrentHashMap**

### (1) 整体结构
&emsp;&emsp;整体的思路其实来源于`HashMap`的线程安全问题只存在于冲突的值，也就是相同hash值的那一个链表的值，因此对整体的操作加锁实际上是没有必要的，`ConcurrentHashMap`就是利用了这一点，把`HashMap`中的数组分割成多个`Segment`，然后对各个`Segment`进行加锁，保证了线程安全并且提高了吞吐量。<br>
&emsp;&emsp;`ConcurrentHashMap`的结构如图所示:(原图作者https://javadoop.com/post/hashmap)
![ConcurrentHashMap结构]({{ '/images/blog/3-1.png' | relative_url }})

### (2) Segment
>Segment类
{:.filename}
{% highlight java linenos%}
    /**
    * Segments是特殊的Hash表，该类继承了ReentrantLock，用来简化锁操作
    */
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        ...
        //存储数据的链表数组，每个HashEntry都是一个链表头
        //volatile来保证线程可见性和顺序性
        transient volatile HashEntry<K,V>[] table;

        //Segment本质就是Hash表，拥有和HashMap几乎相同的操作
        final V put(K key, int hash, V value, boolean onlyIfAbsent) {...}
        final V remove(Object key, int hash, Object value) {...}
        final boolean replace(K key, int hash, V oldValue, V newValue) {...}
        final V replace(K key, int hash, V value) {...}
        final void clear() {...}

        //rehash操作，扩容+数据重新排列
        private void rehash(HashEntry<K,V> node) {...}

        //循环尝试加锁，后面会详细介绍这个方法
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {...}

        //和scanAndLockForPut类似的功能
        private void scanAndLock(Object key, int hash) {...}
        ...        
    }
{% endhighlight %}
&emsp;&emsp;可以看到`Segments`其实就是继承了`ReentrantLock`的一种特殊的Hash表，包含和`HashMap`几乎相同的操作，存储数据使用的是`HashEntry`，每个`HashEntry`包含一个键值对和一个指向下一个`HashEntry`的指针，其结构如下所示：
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
&emsp;&emsp;接下来看看`Segments`是如何保证线程安全性，首先是`put`操作:
>Segments.put()
{:.filename}
{% highlight java linenos%}
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        //尝试获取锁，并利用等待锁的时间创建节点node(节约new对象的时间)，
        //这里的node有两种情况:
        //(1) tryLock()成功，此时无并发，所以不用提前new对象，往下走、
        //(2) tryLock()失败，存在并发，进入scanAndLockForPut重试多次，
        //    在等待并发的时候顺便new一个HashEntry给node
        //当然，上述需要new对象的场景仅发生在key不存在的情况，否则直接更新值就行了。
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;  //计算下标，要存放在哪个链表中
            HashEntry<K,V> first = entryAt(tab, index);  //获取当前链表头
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {  //当前链表存在
                    //这里很简单，就是找到对应的key节点，存入值
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
                else {  //如果找不到对应的key节点，就需要new一个HashEntry
                    if (node != null) //不为null说明在scanAndLockForPut中等待锁的空闲时间已经new好了对象
                        node.setNext(first); //头插法
                    else  //为null说明不存在并发，或者并发很快就结束了，没来得及new对象
                        node = new HashEntry<K,V>(hash, key, value, first); 
                    int c = count + 1;
                    //容量越限，需要扩容+重排列节点
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        setEntryAt(tab, index, node);
                    ++modCount;  //任何修改相关的操作后，都会让modCount++，用于并发下判断我的数据是否被别人动过
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
>Segments.scanAndLockForPut()
{:.filename}
{% highlight java linenos%}
    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;  //node初始为null
        int retries = -1; // 重试次数
        //开始尝试加锁，如果此时并发消失，会直接退出，返回null
        while (!tryLock()) { 
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                //这里很简单，第一次重试失败后，就尝试创建对象
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
            //如果重试多次都不行，就干脆lock来阻塞等待
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            //这里情况是在重试过程中，其他线程修改了该链表，那么就要重头再来
            else if ((retries & 1) == 0 &&
                        (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
{% endhighlight %}

&emsp;&emsp;`Segments`的`remove`，`replace`等操作思路和`put`类似，就不赘述了；奇怪的是`Segments`没有提供`get`操作，实际上`get`是在ConcurrentHashMap中直接实现的，接下来看看ConcurrentHashMap是如何设计的吧：

&emsp;&emsp;（未完待续）