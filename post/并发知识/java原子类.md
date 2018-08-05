### 前言

在JUC包下的的atomic包中，提供了很多线程安全的原子类。在这个包下面，实现线程安全的主要是两种方法。

* 基于CAS的
* 基于累加器的

### 基于CAS的实现

以AcomicInteger为例。通过Unsafe去CAS操作value字段来实现。比较简单，不多bb。

### 基于累加器的实现

Striped64.

>Striped64的设计思路是在竞争激烈的时候尽量分散竞争，在实现上，Striped64维护了一个base Count和一个Cell数组，计数线程会首先试图更新base变量，如果成功则退出计数，否则会认为当前竞争是很激烈的，那么就会通过Cell数组来分散计数，Striped64根据线程来计算哈希，然后将不同的线程分散到不同的Cell数组的index上，然后这个线程的计数内容就会保存在该Cell的位置上面，基于这种设计，最后的总计数需要结合base以及散落在Cell数组中的计数内容


Cell类代码如下

```
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

我们看下longAccumulate的代码。

```
 final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```

代码比较长，逻辑也稍微复杂点，

* cells不为null
	* 需要用到的那个cell对象，没初始化
	* 对cell对象的值进行cas操作，操作成功的话，退出
	* 扩容等操作	 
* cells为null，并且cellBusy不再竞争状态，并且设置为竞争常态成功
	* 初始化cell数组，并根据当前线程所请求的值，创建cell，加入到cells中，退出。 
* 直接cas设置base值
	* 设置成功，退出循环 
	
	
Striped64这个类为基于累加器的实现提供了一些方法，具体的如LongAdder这些的add方法是这样的

```
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```


就不多说了。。

