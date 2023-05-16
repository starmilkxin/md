- [AbstractQueuedSynchronizer(AQS)](#abstractqueuedsynchronizeraqs)
- [ReentrantLock](#reentrantlock)
  - [同步器](#同步器)
  - [非公平锁的实现](#非公平锁的实现)
  - [公平锁的实现](#公平锁的实现)
  - [可重入实现原理](#可重入实现原理)
  - [可打断实现原理](#可打断实现原理)
  - [条件变量实现原理](#条件变量实现原理)
- [自定义锁](#自定义锁)

# AbstractQueuedSynchronizer(AQS)
![AQS继承关系](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326104642.png)

AbstractQueuedSynchronizer是阻塞式锁和相关的同步器工具的框架。<br/>
通过继承该抽象类，我们可以非常便捷地实现我们自己的同步器，以此来生成自定义锁。<br/>
<br/>
AQS 核心原理
- 如果被请求的共享资源未被占用，将当前请求资源的线程设置为独占线程，并将共享资源设置为锁定状态
- AQS 使用一个 Volatile 修饰的 int 类型的成员变量 State 来表示同步状态，修改同步状态成功即为获得锁
- Volatile 保证了变量在多线程之间的可见性，修改 State 值时通过 CAS 机制来保证修改的原子性
- 如果共享资源被占用，需要一定的阻塞等待唤醒机制来保证锁的分配，AQS 中会将竞争共享资源失败的线程添加到一个变体的 CLH 队列中

![AQS原理](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326103732.png)

<br/>
CLH队列
- CLH 队列是一个单向链表，保持 FIFO 先进先出的队列特性
- 通过 tail 尾节点（原子引用）来构建队列，总是指向最后一个节点
- 未获得锁节点会进行自旋，而不是切换线程状态
- 并发高时性能较差，因为未获得锁节点不断轮训前驱节点的状态来查看是否获得锁

![CLH队列](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326105343.png)

<br/>
变体的CLH队列
- AQS 中队列是个双向链表，也是 FIFO 先进先出的特性
- 通过 Head、Tail 头尾两个节点来组成队列结构，通过 volatile 修饰保证可见性
- Head 指向节点为已获得锁的节点，是一个虚拟节点，节点本身不持有具体线程
- 获取不到同步状态，会将节点进行自旋获取锁，自旋一定次数失败后会将线程阻塞，相对于 CLH 队列性能较好

![变体的CLH队列](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326105438.png)

AQS 还继承了AbstractOwnableSynchronizer这个抽象类，而AbstractOwnableSynchronizer则是提供了获取和设置当前独占锁的拥有者线程的方法。<br/>
<br/>
AOS源码如下：

```Java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /** Use serial ID even though all fields transient. */
    private static final long serialVersionUID = 3737899427754241961L;

    /**
     * Empty constructor for use by subclasses.
     */
    protected AbstractOwnableSynchronizer() { }

    /**
     * The current owner of exclusive mode synchronization.
     */
    private transient Thread exclusiveOwnerThread;

    /**
     * Sets the thread that currently owns exclusive access.
     * A {@code null} argument indicates that no thread owns access.
     * This method does not otherwise impose any synchronization or
     * {@code volatile} field accesses.
     * @param thread the owner thread
     */
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    /**
     * Returns the thread last set by {@code setExclusiveOwnerThread},
     * or {@code null} if never set.  This method does not otherwise
     * impose any synchronization or {@code volatile} field accesses.
     * @return the owner thread
     */
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

<br/>
我们来看看AQS中比较重要的方法：<br/>
<br/>
首先是最一开始的上锁acquire方法：

```Java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

接下我们注意看一看内部的每个方法，最后再回到acquire方法中。<br/>
值得一提的是，acquire是不可打断模式噢，之后还有可打断模式的上锁。<br/>
<br/>
tryAcquire是需要自定义的同步器继承后重写的尝试获取锁方法。<br/>
<br/>
addWaiter方法：

```Java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

增加阻塞的线程结点。首先将当前线程封装成结点，之后获取尾部结点。<br/>
如果非空，则将当前线程结点的前置结点设为尾结点。并通过CAS将尾结点设置为node，之后将前置结点(之前的尾结点)的后置结点设置为node，并返回node(当前尾结点)。<br/>
如果尾结点为空亦或是CAS失败，则进入enq方法。<br/>
<br/>

enq方法：

```Java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

内部是for死循环，也就是自旋锁。获取尾结点，如果尾结点为空，则初始化头节点，并将尾结点指向头节点。如果尾结点不为空，则将node结点的前置结点设为尾结点，CAS将尾结点设置为node结点。<br/>
如果CAS成功，则将尾结点的后置结点设置为node结点并返回。<br/>
如果CAS失败，则返回循环自旋，直到成功。<br/>
<br/>
acquireQueued方法：

```Java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

线程添加到阻塞队列后，再次尝试获取锁。首先获取当前node结点的前置结点p。如果p是头结点，并且尝试获取锁成功，则将node设置为头结点，结束返回(注意此处的返回就代表获取锁成功，返回的结果只是表示是被interrupt打断还是被unpark唤醒获得的锁)。<br/>
如果前置结点不为头结点亦或是尝试获取锁失败，则调用shouldParkAfterFailedAcquire方法和parkAndCheckInterrupt方法。<br/>
<br/>
shouldParkAfterFailedAcquire方法：<br/>
首先看看waitStatus的具体数值含义:

```Java

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
```

```Java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

首先获取前驱结点的waitStatus，如果为Node.SIGNAL，则说明当前结点在等待其前驱结点的唤醒。如果大于0，则说明前驱结点被删除了，队列需要调整。其他情况都要强制将前驱结点的waitStatus改为Node.SIGNAL。<br/>
如果为Node.SIGNAL,则返回true，继续parkAndCheckInterrupt方法。<br/>
其余情况都返回false，在acquireQueued方法中继续自旋。<br/>
<br/>

parkAndCheckInterrupt方法：

```Java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

首先是对当前线程进行park阻塞。<br/>
当被唤醒后，通过Thread.interrupted()返回打断的状态。<br/>
注意！如果在被park阻塞时，被interrupt打断的话，那么打断状态是会被保留的(Thread.interrupted()为true)，但是如果是通过unpark唤醒的话，那么Thread.interrupted()为false。<br/>
另，Thread.interrupted()在判断是否被打断时会清空打断状态。

``` Java
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

所以如果是被interrupt打断，那么就返回true(此时打断状态被清空)。<br/>
如果是被unpark唤醒，那么就返回false。<br/>
<br/>
再次回到acquireQueued方法中：

```Java
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
```

如果parkAndCheckInterrupt()返回false，代表是被unpark唤醒，所以标志interrupted就不为true。反之，则设置为true。这为接下来再次自旋获取到锁后返回interrupted做准备。<br/>
<br/>
最后我们再次回到acquire方法重新梳理一遍：

```Java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

首先是tryAcquire方法(需要继承后重写该方法，否则直接抛异常)尝试获取锁。
- 如果获取成功，则方法结束。
- 如果获取失败，则运行&&后的acquireQueued方法中的addWaiter方法。在阻塞队列中添加当前线程结点。

之后运行acquireQueued方法，尝试在队列中获取锁。
- 如果返回true，则说明阻塞的线程是被interrupt打断后才获取到锁的，所以要用selfInterrupt方法恢复线程打断状态，因为parkAndCheckInterrupt中的Thread.interrupted()会清空打断状态。
- 如果返回false，则说明线程未被interrupt打断，是通过unpark唤醒的，也就不用selfInterrupt恢复线程打断状态了。

<br/>
接下来再看看release方法：

```Java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

可以看到，首先尝试解锁(需要继承该类后重写tryRelease)，失败则直接返回false。<br/>
成功，则获取头结点h。如果h为空或h的waitStatus为0(代表头结点的后继节点无需被唤醒)，则直接返回true。反之，则进入unparkSuccessor方法。<br/>
<br/>
unparkSuccessor方法：

```Java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

首先将头结点node的waitStatus用CAS设置为0。之后获取node的后继结点s，如果s为空或者状态是要删除，则从队列尾部倒叙寻找不为null且不为node的结点，如果结点的状态小于等于0，则赋值给s。<br/>
最后如果s不为空，则唤醒s结点中的线程。

# ReentrantLock
![ReentrantLock类图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326195802.png)

首先是synchronized与ReentrantLock的区别

![synchronized与ReentrantLock的区别](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326105743.png)

锁被分为独占锁和共享锁，而独占锁中又分为公平锁和非公平锁。<br/>
ReentrantLock就是一种独占锁，同时它也是可重入锁。<br/>
<br/>
ReentrantLock默认实现非公平锁，你也可以通过参数创建公平锁

```Java
    public ReentrantLock() {
        sync = new NonfairSync();
    }
	
	public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

## 同步器
ReentrantLock中自定义的同步器源码如下：

```Java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```

<br/>
我们来逐个看看重要的方法：<br/>
<br/>
首先是nonfairTryAcquire：

```Java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

首先getState()获取当前锁的状态。<br/>
如果是0，那么就说明锁是空闲的，此时就可以CAS。如果CAS成功，就将独占锁线程拥有者设置为当前线程，并返回true，反之返回false。<br/>
如果c不是0，那么就会有两种情况，独占锁拥有者为本线程和非本线程。当是非本线程时，直接返回false。而是本线程时，就在原来的状态上加上acquries参数(可重入锁)。当加后的nextc小于0，那就是超过最大值。最后将状态值设置好后放回true。<br/>
<br/>
tryRelease方法：

```Java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

首先计算解锁后的状态。然后判断当前线程是否持有锁。如果状态为0，那就说明锁空闲，所以就将ExclusiveOwnerThread设置为null。最后设置好锁状态，如果锁空间就返回true，反之返回false。

## 非公平锁的实现
首先是lock加锁

```Java
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

如果CAS成功，则直接设置好独占锁拥有者线程即可。

![CAS成功](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326214622.png)

<br/>
如果失败，则调用AbstractQueuedSynchronizer中的acquire方法。而acquire方法又首先会调用tryAcquire方法，tryAcquire方法中又是调用nonfairTryAcquire方法。<br/>

```Java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
	
	protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
    }
	
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

如果此时之前的线程释放了锁，锁空闲(c为0)并且CAS设置线程状态成功，亦或是重入锁，则返回true，直接获取锁。<br/>
否则返回false，于是进入接下来的acquireQueued(addWaiter(Node.EXCLUSIVE), arg)。<br/>
<br/>
addWaiter(Node.EXCLUSIVE)在AQS源码中我们是清楚的，就是将当前持有锁的线程封装成结点后插入到阻塞队列中。

```Java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

![addWaiter](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326221024.png)

<br/>
之后则会在acquireQueued自旋通过tryAcquire尝试获取锁，成功则返回。失败，则通过shouldParkAfterFailedAcquire设置前驱结点状态。

```Java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

![shouldParkAfterFailedAcquire](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326221803.png)

<br/>
因为当前结点第一次进入阻塞队列，所以前置结点状态未设置为-1(表示当前结点阻塞，需要前置结点唤醒)，所以返回flase，于是再次在acquireQueued中自旋tryAcquire尝试获取锁。<br/>
如果此时还是失败，则再次进入shouldParkAfterFailedAcquire方法，但是此时因为前驱结点状态为-1，所以返回true，然后接着parkAndCheckInterrupt方法，阻塞当前线程。

```Java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

![parkAndCheckInterrupt](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326222427.png)

<br/>
接下来就是解锁的步骤了，unlock方法调用AQS的release，而AQS的release又会调用自定义sync中重写的tryRelease方法

```Java
    public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
	
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

可以看到tryRelease就是修改线程拥有的锁的状态，如果修改后为0，则说明锁空闲，赋值后，返回true，反之则赋值后返回false。<br/>
tryRelease返回false，release方法就会失败，并返回false。<br/>
tryRelease返回true，并且头结点h存在而且状态值不是0(代表其后继结点阻塞了，需要被唤醒)，就进入unparkSuccessor方法唤醒其后继结点，unparkSuccessor方法也是在AQS源码中就叙述过。<br/>
唤醒成功后，则继续在acquireQueued中尝试获取锁，如果获取成功则如下图:

![获取成功](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220326223908.png)

此时还有可能会有其他线程来争抢锁(非公平锁)，所以如果获取失败就又要回到队列中去了。

## 公平锁的实现
非公平锁和公平锁的差别，就是在尝试获取锁tryAcquire方法的不同。<br/>
<br/>
非公平锁：

```Java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
		
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

<br/>
公平锁：

```Java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

<br/>
可以清楚地看到，二者的差别，只是在于公平锁相较于非公平锁，在CAS改变锁的状态前，有!hasQueuedPredecessors()方法。<br/>
<br/>
hasQueuedPredecessors()方法：

```Java
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

队列中是否含有前驱结点。<br/>
h != t如果返回false，则说明队列中没有node，那么肯定可以进行CAS获取锁操作。hasQueuedPredecessors返回false。<br/>
如果h != t返回true，则说明队列中有node。此时如果(s = h.next) == null返回true，则说明队列中有node，且头结点的后继结点为null，发生了断裂，所以无法进行CAS获取锁。hasQueuedPredecessors返回true。<br/>
如果h != t返回true，(s = h.next) == null返回false，则说明队列中有node，且老二存在。此时如果s.thread != Thread.currentThread())返回true，则说明是其他非老二结点中的线程来争夺锁，因此无法进行CAS操作。hasQueuedPredecessors返回true。<br/>
如果h != t返回true，(s = h.next) == null返回false，s.thread != Thread.currentThread())返回false，说明队列中有node，老二存在，且竞争锁的线程就是老二结点中的线程，完美符合公平的规则，所以可以进行CAS操作。hasQueuedPredecessors返回false。

## 可重入实现原理
无论是公平锁还是非公平锁，它们重写的tryAcquire中的这么一段代码都非常直观地说说明了可重入的原理：

```Java
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
```

可以看到，上锁失败后，如果锁的拥有者线程就是当前线程本身，那么就直接在锁的状态上加上acquires即可，表示锁重入。<br/>
<br/>
它们公共的tryRelease方法也是如此：

```Java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

就在锁状态上减去releases，直到锁状态为0，表示锁空闲。

## 可打断实现原理
ReentrantLock可以选择使用可打断模式的lock(普通的lock和tryLock都是不可打断模式)

```Java
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```

<br/>
nonfairTryAcquire是AQS中已经写好的方法，其中主要是调用到了doAcquireInterruptibly方法，源码如下：

```Java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
	
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

可以看到在acquireInterruptibly方法中，判断如果线程有打断标记，直接抛出异常。之后也是同样先!tryAcquire(arg)尝试获取锁，如果失败，则继续doAcquireInterruptibly方法。<br/>
在doAcquireInterruptibly方法中囊括了addWaiter和acquireQueued方法，最重要的区别就是，在判断parkAndCheckInterrupt()返回true(表示是被interrupt打断而唤醒的)后，直接抛出打断异常。<br/>
<br/>
总结:
- 可打断模式，线程在阻塞时被interrupt打断就直接抛出异常，只有被unpark唤醒才会继续自旋尝试获取锁。
- 不可打断模式，线程在阻塞时无论是被interrupt打断还是unpark唤醒，之后都会继续自旋尝试获取锁。

## 条件变量实现原理
每个条件变量其实就对应着一个等待队列，其实现类是 ConditionObject。<br/>
ConditionObject是AQS类中的一个类，并且实现了Condition接口。<br/>
<br/>
首先是ReentrantLock中的newCondition方法：

```Java
    public Condition newCondition() {
        return sync.newCondition();
    }
```

<br/>
然后是继承了AQS的Sync类中的newCondition方法：

```Java
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
```

所以就是创建了一个AQS类中的ConditionObject类的对象。<br/>
<br/>
接下来是await方法：

```Java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

我们来挨个看看其中的每个方法。<br/>
<br/>
addConditionWaiter方法:

```Java
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

将结点添加到条件变量队列中。首先获取条件变量队列中的最后结点。如果不为空但是状态不是CONDITION的话，就解锁已经删除的结点，并让结点t为最后结点。<br/>
将当前线程封装成结点node。如果最后结点为null，那么node就是第一个结点。否则，node是最后一个结点的后继结点。<br/>
最让再让最后结点为node，返回node。<br/>
<br/>
fullyRelease方法：

```Java
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

完全释放锁。首先获取锁状态，然后完全释放锁，saveState大于1时，就说明有锁重入情况产生，此时可以实现完全释放。之后返回，并将自己结点设置为已删除。<br/>
<br/>
isOnSyncQueue方法：

```Java
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }
```

isOnSyncQueue判断线程结点是否在同步队列中。<br/>
<br/>
checkInterruptWhileWaiting方法：

```Java
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```

判断被唤醒是通过unpark还是interrupt，如果是unpark，那么返回0，否则，返回非0。<br/>
<br/>
acquireQueued方法之前已经讲述过。<br/>
<br/>
接下来我们看看signal方法：

```Java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

当条件队列首结点不为空，则调用doSignal方法。<br/>
<br/>
doSignal方法：

```Java
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

先令first的下个结点成为firstWaiter，之后再令first自己的下一个结点为null，这样就可以离开条件变量队列了。<br/>
之后执行transferForSignal方法。<br/>
<br/>
transferForSignal方法：

```Java
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

首先通过CAS试图将结点的状态从CONDITION改为0，如果失败，并且firstWaiter不为空，则在doSignal中继续自旋。<br/>
执行enq方法，将node插入到阻塞队列尾部，接下来如果取消或尝试设置node的前驱结点p的waitStatus失败，则唤醒node以重新同步。<br/>
<br/>
整体流程过一遍：<br/>
首先是await阻塞：

```Java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

首先将当前线程结点，插入到条件变量队列的尾部。之后完全释放所占有的锁，因为自己不在同步队列中，所以被park阻塞。

![await](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220327202916.png)

<br/>
此后当Thread1占有锁后，并且唤醒条件变量队列中的Thread0时：

```Java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
		
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
		
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

首先将结点从条件变量队列中删除，然后将其结点状态改变后，插入到阻塞队列的尾部。<br/>
如果它的前置结点被删除亦或是无法将其状态改为SIGNAL的话，就直接唤醒node。<br/>
反之，则直接返回true，因为前驱结点状态设置成了SIGNAL，所以迟早自己在同步队列中也会被unpark唤醒，于是在await方法中就可以被唤醒，又因为是在同步队列中，就可以退出循环，转而调用acquireQueued(node, savedState)。这个方法我们是熟悉的，就是在同步队列中尝试获取锁。

![signal](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220327203112.png)

# 自定义锁

```Java
//自定义锁(不可重入)
class MyLock implements Lock {

    //独占锁同步器
    class MySync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                //将独占锁的拥有者线程设置为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            //因为state有volatile修饰，所以设置拥有者线程要放在state前，保证可见性
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        public Condition newCondition() {
            return new ConditionObject();
        }
    }

    //同步器
    private MySync sync = new MySync();

    //加锁(不成功则到等待队列进行等待)
    @Override
    public void lock() {
        sync.acquire(1);
    }

    //可打断加锁
    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    //尝试加锁(尝试一次)
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }


    //带超时的尝试加锁
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    //解锁
    @Override
    public void unlock() {
        sync.release(1);
    }

    //创建条件变量
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

<br/>

参考：
