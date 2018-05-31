### 前言


在JDK中，优先级队列使用小顶堆实现的。在此之前，我们需要先了解点基础知识。

* 完全二叉树：若设二叉树的深度为h，除第 h 层外，其它各层 (1～h-1) 的结点数都达到最大个数，第 h 层所有的结点都连续集中在最左边，这就是完全二叉树。
* 小顶堆：是一种经过排序的完全二叉树，其中任一非终端节点的数据值均不大于其左子节点和右子节点的值

![](https://gss1.bdstatic.com/-vo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=7ee311d70923dd54357eaf3ab060d8bb/b151f8198618367a6f44126e2e738bd4b21ce5b0.jpg)

如果我们用数组去表示完全二叉树的话。就有以下特点，

* leftNo = parentNo*2+1

* rightNo = parentNo*2+2

* parentNo = (nodeNo-1)/2



接下来关于小顶堆的操作，就看源码了。

### offer

```
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
```

在添加数据的时候，会调用siftUp去调整位置，位置小顶堆特点。


![](https://images2015.cnblogs.com/blog/939998/201605/939998-20160512205600890-346195840.png)


原理比较简单，就是放置在最后，和自己父做比较，如果大于父，则交换

```
    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```

### poll

```
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);
        return result;
    }
```

![](https://images2015.cnblogs.com/blog/939998/201605/939998-20160512205634609-402016454.png)


```
    @SuppressWarnings("unchecked")
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo((E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
```

代码不难，不解释了。
