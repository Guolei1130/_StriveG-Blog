### 前言

EnumMap是一种比较特殊的Map，原因就在于他的key只能是Enum枚举类型。

EnumMap的源码实现也比较简单，并且没有扩容机制。接下来，从下面几个方面，简单的介绍下。

### 构造函数

以下面这个构造函数为例。


```
    public EnumMap(Class<K> keyType) {
        this.keyType = keyType;
        keyUniverse = getKeyUniverse(keyType);
        vals = new Object[keyUniverse.length];
    }
```

在构造函数中，置顶了keyType，并且获取keyUniverse并赋值，这个keyUniverse是什么呢？就是枚举中的几个枚举量。并且构造一个数组来存放相应枚举量对应的value。

### get方法。

```
    public V get(Object key) {
        return (isValidKey(key) ?
                unmaskNull(vals[((Enum<?>)key).ordinal()]) : null);
    }
```

get方法也比较简单，主要分为两步，首先校验key的合法性，其次，直接给去key所对应的枚举量的索引值去vals数组中取出相应的值即可。


### put方法

```
    public V put(K key, V value) {
        typeCheck(key);

        int index = key.ordinal();
        Object oldValue = vals[index];
        vals[index] = maskNull(value);
        if (oldValue == null)
            size++;
        return unmaskNull(oldValue);
    }

```

根据key在枚举中的索引值，将value存放在vals数组的相应位置。

