fail-fast和fail-safe的区别：
+ fail-safe允许在遍历的过程中对容器中的数据进行修改，而fail-fast则不允许。

# fail-fast
在容器中使用迭代器遍历的过程中，一旦发现容器中的数据被修改了，会立刻抛出ConcurrentModificationException异常导致遍历失败。java.util包下的集合类基本都是快速失败机制的, 常见的容器有HashMap和ArrayList等。<br/>
<br/>
比如：

```Java
        HashMap<Integer, Integer> hashMap = new HashMap<>();
        hashMap.put(1, 1);
        hashMap.put(2, 3);
        Iterator<Map.Entry<Integer, Integer>> iterator = hashMap.entrySet().iterator();
        while (iterator.hasNext()) {
            hashMap.remove(1);
            iterator.next();
        }
		
/*结果
Exception in thread "main" java.util.ConcurrentModificationException
*/
```

具体的原理很简单：<br/>
无论是HashMap还是ArrayList等容器，当有增加、减少、修改元素操作时都会有：

```Java
        ++modCount;
```

<br/>
而在iterator.next()时：

```Java
    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }
	
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
			//关键就在以下这句
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
```

迭代器在初始化时，会使得modCount和expectedModCount相等。<br/>
可以看到，每次在迭代器遍历前，都会判断modCount != expectedModCount。如果二者不相等，那么肯定在迭代器遍历的期间发生了，增删改操作，所以就直接抛出异常。<br/>
<br/>
但是也有办法可以防止这种情况：
+ 不适用迭代器进行遍历。
+ 使用ConcurrentHashMap或者CopyOnWriteArrayList。
+ 使用迭代器自身的remove()方法。

```Java
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
```

虽然iterator.remove()，在removeNode方法中也会令modCount加1，但是在remove方法最后，会令expectedModCount = modCount。因此迭代器遍历时不会有问题。

# fail-safe
这种遍历基于容器的一个克隆。因此，对容器内容的修改不影响遍历。java.util.concurrent包下的容器都是安全失败的,可以在多线程下并发使用,并发修改。常见的的使用fail-safe方式遍历的容器有ConcerrentHashMap和CopyOnWriteArrayList等。<br/>
<br/>
采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。<br/>
<br/>
参考：
+ https://blog.csdn.net/striner/article/details/86375684