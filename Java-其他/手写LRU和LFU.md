# LRU

最近最少使用

leetcode 146

```java
class LRUCache {

    class Node {
        int key;
        int value;
        Node next;
        Node pre;

        public Node() {
        }

        public Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    Node dummyFirst;

    Node dummyLast;

    int size;

    Map<Integer, Node> map;

    int capacity;

    public LRUCache(int capacity) {
        map = new HashMap<>();
        this.capacity = capacity;
        dummyFirst = new Node();
        dummyLast = new Node();
        dummyFirst.next = dummyLast;
        dummyLast.pre = dummyFirst;
    }

    public void addLast(Node node) {
        dummyLast.pre.next = node;
        node.pre = dummyLast.pre;
        node.next = dummyLast;
        dummyLast.pre = node;
        this.size++;
        map.put(node.key, node);
    }

    public void removeFirst() {
        Node first = dummyFirst.next;
        dummyFirst.next = dummyFirst.next.next;
        dummyFirst.next.pre = dummyFirst;
        this.size--;
        map.remove(first.key);
    }
    
    public void removeNode(Node node) {
        node.pre.next = node.next;
        node.next.pre = node.pre;
        this.size--;
        map.remove(node.key);
    }

    public int get(int key) {
        Node node = map.get(key);
        if (node == null) {
            return -1;
        }
        int value = node.value;
        refreshNode(node);
        return value;
    }
    
    public void put(int key, int value) {
        Node node = map.get(key);
        if (node == null) {
            node = new Node(key, value);
            if (this.size == capacity) {
                removeFirst();   
            }
            addLast(node);
        } else {
            node.value = value;
            refreshNode(node);
        }
    }

    public void refreshNode(Node node) {
        if (this.dummyLast.pre == node) {
            return;
        }
        removeNode(node);
        addLast(node);
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```


# LFU

最不经常使用

leetcode 460

![LFU](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/LFU.png)

```java
class LFUCache {

    class Node {
        int key;
        int value;
        int fre;

        public Node() {
        }

        public Node(int key, int value) {
            this.key = key;
            this.value = value;
            this.fre = 1;
        }
    }

    Map<Integer, Deque<Node>> mapFre;

    Map<Integer, Node> mapKV;

    int capacity;

    int size;

    int minFre;

    public LFUCache(int capacity) {
        this.capacity = capacity;
        mapFre = new HashMap<>();
        mapKV = new HashMap<>();
    }
    
    public void insertNode(Node node) {
        if (this.minFre == 0 || node.fre < this.minFre) {
            this.minFre = node.fre;
        }
        Deque<Node> dq = mapFre.get(node.fre);
        if (dq == null) {
            dq = new LinkedList<>();
            mapFre.put(node.fre, dq);
        }
        dq.addLast(node);
        mapKV.put(node.key, node);
        this.size++;
    }

    public void removeNode() {
        if (this.size == 0) {
            return;
        }
        Deque<Node> dq = null;
        while (true) {
            dq = mapFre.get(this.minFre);
            if (dq == null) {
                this.minFre++;
            } else {
                break;
            }
        }
        Node rmNode = dq.removeFirst();
        mapKV.remove(rmNode.key);
        this.size--;
        if (dq.size() == 0) {
            mapFre.remove(this.minFre);
            this.minFre++;
        }
    }

    public void removeNode(Node node) {
        Deque<Node> dq = mapFre.get(node.fre);
        if (dq == null) {
            return;
        }
        dq.remove(node);
        mapKV.remove(node.key);
        this.size--;
        if (dq.size() == 0) {
            mapFre.remove(node.fre);
            if (node.fre == this.minFre) {
                this.minFre++;
            }
        }
    }

    public int get(int key) {
        Node node = mapKV.get(key);
        if (node == null) {
            return -1;
        }
        int value = node.value;
        removeNode(node);
        node.fre++;
        insertNode(node);
        return value;
    }
    
    public void put(int key, int value) {
        Node node = mapKV.get(key);
        if (node == null) {
            node = new Node(key, value);
            mapKV.put(key, node);
            if (this.size == this.capacity) {
                removeNode();
            }
        } else {
            removeNode(node);
            node.value = value;
            node.fre++;
        }
        insertNode(node);
    }
}

/**
 * Your LFUCache object will be instantiated and called as such:
 * LFUCache obj = new LFUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```