---
layout: post
title: weak reference
date: 2015-06-30 23:06:56
tags: java
---

##1

最近看了一篇文章[Why 0x61c88647?](http://www.javaspecialists.eu/archive/Issue164.html)，感觉写的很细致。

关于`ThreadLocal`的问题，这里就不复述了。

文章在讲解`ThreadLocal`的实现的时候，提到了`WeakReference`。

感觉还是挺陌生的，印象中在用`Guava`的`Cache`的时候遇到过设置`key`和`value`为`weak`状态。（不过使用的时候也没配置过相关的选项……）

大致了解是跟GC有关系，其它就不清楚了，所以就多google一下吧。

##2

[Understanding Weak References](https://weblogs.java.net/blog/2006/05/04/understanding-weak-references)

找到这篇文章，感觉介绍的挺直观。

简单总结一下：

通常的Java引用是strong reference。但是对于有些场景，比如做为缓存的`Map`，需要维护增减的键值，就有点类似于手动地分配、释放内存，这就失去自动垃圾回收的优势了嘛。

所以就可以使用`WeakHashMap`，其中的键是弱引用的，GC就可以回收到，顺便值也自动回收了。

另外，“弱”的程度：

Soft references：weak和strong是发现了就回收，而soft得等等。

Phantom references：感觉使用场景想不太到。

##3

看一下`WeakHashMap`的实现吧~

```java
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> {

    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    Entry<K,V>[] table;
    
    /**
     * Reference queue for cleared WeakEntries
     */
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
    
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for this key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated.
     * @param value value to be associated with the specified key.
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        Object k = maskNull(key);
        int h = hash(k);
        Entry<K,V>[] tab = getTable();
        int i = indexFor(h, tab.length);

        for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
            if (h == e.hash && eq(k, e.get())) {
                V oldValue = e.value;
                if (value != oldValue)
                    e.value = value;
                return oldValue;
            }
        }

        modCount++;
        Entry<K,V> e = tab[i];
        tab[i] = new Entry<>(k, value, queue, h, e);
        if (++size >= threshold)
            resize(tab.length * 2);
        return null;
    }
    
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Object k = maskNull(key);
        int h = hash(k);
        Entry<K,V>[] tab = getTable();
        int index = indexFor(h, tab.length);
        Entry<K,V> e = tab[index];
        while (e != null) {
            if (e.hash == h && eq(k, e.get()))
                return e.value;
            e = e.next;
        }
        return null;
    }
    
    /**
     * Returns the table after first expunging stale entries.
     */
    private Entry<K,V>[] getTable() {
        expungeStaleEntries();
        return table;
    }
    
    /**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
    
    /**
     * The entries in this hash table extend WeakReference, using its main ref
     * field as the key.
     */
    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;
        
        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
    }
}
```

逻辑非常清楚——一个通过链接法解决冲突的散列表。

其中有两个地方比较关键：

第一个，`Entity`的构造方法，传入了`ReferenceQueue`，这样当某个`WeakReference`被回收的时候将会进入队列；

第二个，`expungeStaleEntries()`，清除散列表中被回收的key，同时将value设为空（这个对GC帮助很大呀）。

