### 前言

ArrayDeque的使用频率相对比较低。他实现了Deque接口，是双端队列的数组实现方式。ArrayDeque的实现十分的有意思。下面就来分析一下。首先，有两个int类型的变量head和tail充当两个指针，分别指向队首和队尾。接下来分析一些其中的方法。


### 构造方法

无参的构造方法如下。

```
    public ArrayDeque() {
        elements = new Object[16];
    }
```

可以看到默认的大小是16。

而传入容量的那个构造方法如下。

```
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }
    
        private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        elements = new Object[initialCapacity];
    }
```

这个的大小则是比指定容量大于等于的2的n次方的数。

### 添加数据

添加数据的方法很多，包括双端队列接口的一些方法以及Collection接口的一些方法，但是这个方法都是通过addFirst以及addLast这两个方法添加数据的，我们分别看下。

addFirst

```
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;
        if (head == tail)
            doubleCapacity();
    }
```

从源码中我们知道，ArrayDeque是不允许null值的。当我们添加第一个数据是。head为初始值0,则(head-1) & (elements.length-1)为15（默认16容量）,插入在数组的最后一个，再插入是，为14，是倒数第二个，可以看出，addFirst的实现，是从数组的后面往前面插入。

addLast

```
    public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e;
        if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            doubleCapacity();
    }
```

而addLast的方法则是从前面往后面。


### 扩容机制

当数组中没有可用容量的时候，会进行扩容。其实现为doubleCapacity方法。

```
    private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    }    
```

这便是ArrayDeque中比较有意思的代码。逻辑非常清晰。将原数组中的数据全部复制到新数组中，并且对head和tail重新赋值。这样，再插入的时候 就相当于把扩容部分的子数组当做一个全新的数组，和上面的添加数据的方法就没区别了。但是前半部分的子数组(0,n)确实按照从前到后的add顺序排列的。

### 总结

虽然只介绍了一小部分的内容，但是，当我们理解了他的添加数据以及扩容机制之后，其他的方法已经不在是难点。总的来说，ArrayDeque的实现也不难，但是却非常的巧妙。
