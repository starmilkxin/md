# ReentrantReadWriteLock
读写锁的获取

```Java
ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
ReentrantReadWriteLock.ReadLock r = lock.readLock();
ReentrantReadWriteLock.WriteLock w = lock.writeLock();
```

<br/>
ReentrantReadWriteLock中的Sync用前十六位表示独占锁的数量，而后十六位表示共享锁的数量：

```Java
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

<br/>
首先我们来看看读锁/共享锁的上锁lock：

```Java
        public void lock() {
            sync.acquireShared(1);
        }
		
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

可以看到最后是先调用了tryAcquireShared，再调用了doAcquireShared。<br/>
<br/>
tryAcquireShared方法：

```Java
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

<br/>
其中的readerShouldBlock在公平锁和非公平锁中还不一样：

```Java
//非公平锁
final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
		
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }

//公平锁
final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
}
```

公平锁要前面无结点才行。而非公平锁，则是要判断头结点的后继结点是不是写锁，如果是写锁则返回true(避免写锁无限等待)，否则返回false。<br/>
<br/>
之后就会判断r是不是小于最大值，CAS设置锁状态值是否成功，成功的话，可以看到，它共享锁的特性就体现出来了：
+ 如果当前共享锁数量为0，那么则firstReader的值为当前线程，并且firstReaderHoldCount变为1.
+ 如果firstReader就是当前线程，那么firstReaderHoldCount++，代表重入锁。最后rh.count++。
+ 如果不是的话，那么就要看看自己有没有cachedHoldCounter，如果没有，或者tid不一样，那么就要从ThreadLocal中去获取。如果有，并且tid一样，但是rh.count为0，那么就要在ThreadLocal中放入当前rh。最后rh.count++。

失败，则执行并返回fullTryAcquireShared(current)，是tryAcquireShared的复杂版。<br/>
<br/>
最后的doAcquireShared方法：

```Java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
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

共享锁的doAcquireShared和独占锁的acquireQueued很像，不过将addWaiter放在doAcquireShared中一起执行了，将结点属性设置为了shared。setHeadAndPropagate还会继续唤醒自己后继的共享的结点。<br/>
<br/>
总结来说：
+ 共享锁如果能上锁，那么就直接用变量存取第一个上锁的线程，并记录数量。其中如果有多个线程上了共享锁，那么就需要ThreadLocal来记录每个线程各自上锁的数量。
+ 共享锁如果不能上锁，那么就和独占锁一样要进入同步队列进行排队等待被唤醒。

<br/>
接下里我们来看看共享锁的释放unlock：

```Java
        public void unlock() {
            sync.releaseShared(1);
        }
		
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

<br/>
tryReleaseShared方法：

```Java
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

如果当前线程就是firstReader的话，就将firstReaderCount减去一。否则就靠ThreadLocal获取自己的HoldCounter然后再减一。<br/>
之后再获取减去一个共享单元后的状态值，自旋通过CAS修改锁的状态值。返回状态值是否为0。<br/>
<br/>
最后的doReleaseShared方法：

```Java
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

逻辑是一个死循环，每次循环中重新读取一次head，然后保存在局部变量h中，再配合if(h == head) break;，这样，循环检测到head没有变化时就会退出循环。注意，head变化一定是因为：acquire thread被唤醒，之后它成功获取锁，然后setHead设置了新head。而且注意，只有通过if(h == head) break;即head不变才能退出循环，不然会执行多次循环。<br/>
<br/>
ReentrantLock默认就是独占锁。所以这里ReentrantReadWriteLock中的独占锁和ReentrantLcok是几乎没啥区别的。
