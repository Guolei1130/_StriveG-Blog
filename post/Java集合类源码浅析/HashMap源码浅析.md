### 前言

HashMap在平常开发中使用频率很高，虽然使用起来很简单，但是这个类的源码确实十分的复杂，尤其是在1.8之后，加入了红黑树，使得其工作原理更加复杂，本篇可能只会简单的介绍其中的一些原理。红黑树部分的代码等到TreeMap再说。

### 理解关键变量

* loadFactor 装载因子。装载因子是个很重要的参数，小了容易造成使用率低，大了的话，添加数据容易造成hash碰撞，影响效率。默认是为0.75
* TREEIFY_THRESHOLD 树化的阀值，当超过这个值是，会将链表转为话红黑树
* UNTREEIFY_THRESHOLD 非树化的阀值，当小于这个值时，将红黑树转为链表


### hash函数

HashMap的好坏，hash函数是一个很重要的性能点，一个好的hash函数应该尽可能是hash分布均匀，减少干扰。看一下java 1.8的hash函数。

```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

hashcode和hashcode的高16位进行异或操作。我们要向理解这个操作，我们还需要知道，hashmap中找桶的操作。

```
tab[i = (n - 1) & hash])
```

从找桶位的操作来看，如果单单靠hashcode的话，hashcode的高位并不会影响到找桶的操作，这样更容易造成冲突，因此，让高16位也参与运算。就形成了上面的hash操作。

### put方法

```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

三种情况:

* table没初始化，先初始化table
* 计算出来的索引处空着，直接插入索引处
* 发生碰撞
	* key的hashcode和值与现在的都相等，进行覆盖
	* 若索引处存的是TreeNode，那么调用putTreeVal像树种插入数据
	* 插入数据到链表，如果达到阀值，进行树化。
	
其实这里的逻辑并不复杂，我个人对hashmap理解不到的主要原因是对红黑树的不了解。

### resize

在对原桶位进行扩容之后，还需要把原来的数据加入进来。问题不大。

### remoe

对于数据链表的结构来说，remove的操作并不复杂。

### 总结

总的来说，这篇HashMap的介绍写的很浅显。但是java 1.8中加入了红黑树，倒是源码变的很复杂，HashMap中有很多细节值得我们学习。



 



