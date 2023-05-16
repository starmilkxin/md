# ThreadLocal流程

![ThreadLocal流程图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/ThreadLocal原理图.png)

如图所示，当你调用ThreadLocal的set方法时，其实是获取到了当前Thread的ThreadLocalMap。

```Java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

<br/>
此后再将当前的ThreadLocal对象作为key，搭配value，一齐插入到当前Thread的threadLocals的某个entry中，作为键值对。

# ThreadLocal内存泄露原因以及如何避免
ThreadLocal内存泄露的原因为：
1. 线程长时间存活
2. 无法通过ThreadLocal调用其remove()方法删除val

针对1，因为Thread线程长时间存活且其是被强引用，所以GC无法回收Thread，因此也无法回收Thread内部的强引用对象ThreadLocalMap，因此也无法回收其内部的强引用对象Entry，因此也无法回收其中的强引用对象Object value，导致内存一直被占有。

针对2，此时其实并不算内存泄露，因为这是开发者自身没有去将value值清空的原因，如果GC可以随便回收value值，那才叫恐怖好吧。。。但是如果我们无法通过ThreadLocal调用remove()方法删除val，这时才是真正的内存泄露！当外部的强引用ThreadLocal被置为null后，那么我们无法调用其remove()方法，因为会发生空指针问题。

网上很多人说ThreadLocal内存泄露的原因是因为其弱引用导致的:

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

但是无论是将ThreadLocal作为弱引用对象，还是强引用对象，当将外部的其强引用置为null后，都是会发生无法调用remove()方法导致内存泄露问题发生的！
如果将其作为强引用对象，那么会造成key和value都无法消除的情况产生，会加剧内存泄漏。
而如果将ThreadLocal作为弱引用对象，那么当外部的强引用为null后，对象所占用的内存便会被回收，entry中的key便为null，此时虽然val依旧存在，会发生内存泄漏，但是起码情况比key没被回收要好！同时也可以根据key是否为null来判断是否要对value进行回收。这也就是为什么ThreadLocal要设置为弱引用对象的原因。
<br/>
解决方法：
+ 主动调用ThreadLocal的remove()方法清除val。
+ 当无法主动调用ThreadLocal的remove()方法时，ThreadLocal的部分操作也会能让我们被动地清除那些entry的key为null所对应的val，下面会说到。

# ThreadLocal部分操作源码
首先是set方法

```Java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

<br/>
可以看到，首先是获取到了当前线程中的ThreadLocalMap，然后调用ThreadLocalMap的set方法，key就是ThreadLocal，val就是value值。
<br/>
接下来是ThreadLocalMap的set方法：

```Java
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

可以看到，首先是计算出key的hashCode，然后根据hashCode在以Entry为元素的数组tab中找到对应的Entry。
如果二者的key相同，那么直接覆盖并返回。
如果key为null，那么会调用replaceStaleEntry(key, value, i)来将该槽位进行覆盖操作，注意此方法内部还会对该槽位前后进行扫描来删除key为null的槽位的val值等其他操作，很复杂！
如果key既不相同，也不为null，那就一直遍历直到槽位为空，退出for循环为止。之后就new 一个 Entry，改变size大小。
之后判断，如果并没有清空其他槽位成功，并且新改变的size比threshold阈值大的话，就需要进行rehash操作（因为size大小改变了）。
<br/>
replaceStaleEntry方法：

```Java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

该方法比较复杂，其主要思想就是，为了解决hash冲突的开放寻址法带来的有多个key相同的槽位问题（一开始A插入到5槽位，后来B也插入到5槽位发生冲突于是开放寻址到7槽位，之后A被删除，又有B插入到5槽位，造成两个key相同的槽位问题）。所以在插入到空槽位时，要分别向前遍历和向后遍历，来删除掉发生hash冲突的槽位。同时还会执行cleanSomeSlots方法来遍历后面的槽位，将key为null的槽位的val也置空，防止内存泄漏发生！
<br/>
cleanSomeSlots方法：

```Java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

通过遍历之后的槽位，来将那些key为null的槽位的val也清空，以此避免内存泄漏。
<br/>
rehash方法：

```Java
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

可以看到rehash会对所有槽位进行一个key是否为null的判别。

<br/>
set方法结束了，接下来是ThreadLocal的get()方法：

```Java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

逻辑比较简单，主要是map.getEntry方法和其中的getEntryAfterMiss的具体实现。

```Java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

当getEntry并没有获取到对应的key时，那么就说明发生了hash冲突，所以要调用getEntryAfterMiss方法，向后去检查对应槽位的key，相等则返回，为空则调用expungeStaleEntry方法进行清空。不相等也不为空，则跳到下一个i继续判别。直到找到返回entry或者找不到返回null。
<br/>
最后就是remove()方法了：

```Java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}

//Reference
public void clear() {
    this.referent = null;
}
```

除了要将referent，也就是key置为null，还要调用expungeStaleEntry方法，将当前槽位的值以及后续槽位key为null的对应的val都置为null。