### 前言

AbstractQueuedSynchronizer 队列同步器，是很多锁和同步组件的基础。这个是基于CLH自旋锁对列来实现的。


### CLH的node节点


```
    static final class Node {

        static final Node SHARED = new Node();

        static final Node EXCLUSIVE = null;


        static final int CANCELLED =  1;

        static final int SIGNAL    = -1;

        static final int CONDITION = -2;
     
        static final int PROPAGATE = -3;

        volatile int waitStatus;

      
        volatile Node prev;

        volatile Node next;

      
        volatile Thread thread;

    
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

       
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
```

节点的状态如下：

* CANCELLED 已经取消
* SIGNAL 当当前release的时候，需要去通知后续节点进行unpark
* CONDITION 在条件队列上
* PROPAGATE releaseShared需要告诉其他节点
* 0 退出了


### acquire方法

```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这个方法是以独占的模式去获取锁。这个方法会阻塞，直到获取资源成功，如果阻塞的过程中发生了终端，那么就会调用selfInterrupt进入中断。tryAcquire由具体的实现类去实现，我们今天不说。我们重点看下。addWaiter和acquireQueued方法

#### addWaiter

```
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

这个方法配合上enq方法，实际上就是创建一个节点放在CLH队列的队尾。

#### acquireQueued


```
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

在这里自旋，
* 一直等到该节点的前驱节点为CLH队列头节点并且tryAcquire成功，设置改节点为node节点，这个时候，就是获取资源成功了。然后就退出了。
* 获取失败后应该挂起，什么时候应该挂起呢，就是改节点的前驱状态为SIGNAL，这个时候park线程，并且返回值为中断。


### release

```
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

如果tryRelease成功，并且CLH头节点不为null且状态不是0的话，就unpark他的后继。

#### unparkSuccessor

```
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

就是找到第一个不是cncelled状态的后继，并且unpark线程即可。


共享模式的话，这里就不说了。emm

