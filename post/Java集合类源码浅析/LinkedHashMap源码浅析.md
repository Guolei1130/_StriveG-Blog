LinkedHashMap相对来说也比较常用。在Android中，LruCache就是用LinkedHashMap来实现最近最少使用策略的。LinkedHashMap是基于HashMap的，LinkedHashMap在HashMap的基础上，增加了head、tail，并扩展HashMap的Node，加载before和after，，在每次访问 插入 删除数据之后，都会根据accessOrder去更改双向链表。接下来，我们就看下部分源码实现吧。


### 插入数据时

虽然在HashMap插入数据的方法里，有afterNodeInsertion,从名字上来看，我们似乎应该关心LinkedHashMap中这个方法。不过，插入数据时区操作链表可不是在这个里面，是在newNode里。如下。

```
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
```

在newNode中会将生成的node，链接到双向链表的尾端。


```
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```

### 删除数据时

删除数据的时候，调用afterNodeRemoval去删除双向链表中的这个节点。

```
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

### 访问数据时

访问数据时的代码如下。


```
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

还是比较简单的。将这个节点的上一个节点的after指向这个节点的下一个节点，将下一个节点的before指向这个节点的上一个，将这个节点加到链表尾端即可。