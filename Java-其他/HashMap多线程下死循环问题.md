jdk1.7及以前，HashMap在多线程并发插入的情况下，有可能会造成死循环导致程序崩溃。<br/>
<br/>
当HashMap中的元素达到阈值后，就会进行扩容，并进行reHash

```Java
        if (++size > threshold)
            resize();
```

<br/>
reHash前的HashMap：

![reHash前的HashMap](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402100955.png)

<br/>
reHash后的HashMap：

![reHash后的HashMap](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402100901.png)

<br/>
<br/>
假设一个HashMap已经到了Resize的临界点。此时有两个线程A和B，在同一时刻对HashMap进行Put操作：

![Put](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103605.png)
![Put2](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402102938.png)

<br/>
此时达到Resize条件，两个线程各自进行Rezie的第一步，也就是扩容：

![resize](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402102955.png)

<br/>
这时候，两个线程都走到了ReHash的步骤。让我们回顾一下ReHash的代码：

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
			//B线程在此处被挂起
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

假如此时线程B遍历到Entry3对象，刚执行完红框里的这行代码，线程就被挂起。<br/>
对于线程B来说：
+ e = Entry3
+ next = Entry2

这时候线程A畅通无阻地进行着Rehash，当ReHash完成后，结果如下（图中的e和next，代表线程B的两个引用）：

![A rehash完](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103018.png)

<br/>
接下来线程B恢复，继续执行属于它自己的ReHash：

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
			//B线程从此处恢复
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
			//B线程执行到此处
        }
    }
}
```

Entry3放入了线程B的数组下标为3的位置，并且e指向了Entry2。此时e和next的指向如下：
+ e = Entry2
+ next = Entry2

整体情况如图所示：

![整体](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103032.png)

<br/>
B线程继续执行：

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
			//B线程执行到此处
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```
此时：
+ e = Entry2
+ next = Entry3

整体情况如图所示：

![再次整体](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103044.png)

<br/>
B线程继续执行：

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
			//B线程执行到此处
        }
    }
}
```

![整体3](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103053.png)

<br/>
第三次循环开始，又执行到红框的代码：

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
			//B线程执行到此处
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

此时
+ e = Entry3
+ next = Entry3.next = null

<br/>
最后:

```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
			//B线程执行到此处
            e = next;
        }
    }
}
```

此时发生了循环：
+ newTable[i] = Entry2
+ e = Entry3
+ Entry2.next = Entry3
+ Entry3.next = Entry2

![循环](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220402103141.png)

此时，问题还没有直接产生。当调用Get查找一个不存在的Key，而这个Key的Hash结果恰好等于3的时候，由于位置3带有环形链表，所以程序将会进入死循环！<br/>
<br/>
参考：
+ [漫画：高并发下的HashMap](https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653192000&idx=1&sn=118cee6d1c67e7b8e4f762af3e61643e&chksm=8c990d9abbee848c739aeaf25893ae4382eca90642f65fc9b8eb76d58d6e7adebe65da03f80d&scene=21#wechat_redirect)