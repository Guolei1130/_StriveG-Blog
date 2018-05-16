HashSet相对来说就比较容易了。他是利用HashMap或者LinkHashMap做存储，


```
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

通过讲E作为map的key来达到不允许重复的目的。

由于这里确实比较简单，没什么说的。就这样吧。