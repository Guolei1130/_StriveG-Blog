### 前言

平常开发中，我们几乎不会用到EnumSet，以至于我们对EnumSet的用法都不熟悉。EnumSet是一个抽象类，内含一些静态方法，来构造一个EnumSet。比如。

* allOf
* copyOf
* 等等

首先，我们看下allof的实现。

```
    public static <E extends Enum<E>> EnumSet<E> allOf(Class<E> elementType) {
        EnumSet<E> result = noneOf(elementType);
        result.addAll();
        return result;
    }
    
    public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum<?>[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }
```


可以看到，EnumSet有两个实现类，一个是RegularEnumSet，这是Enum长度小于等于64的情况下，另一个是大于64的情况。这两个实现类，在实现上差别并不是很大。接下来，我们分别看一下这两个实现的部分方法。


### RegularEnumSet

这种实现，通过一个叫elements的变量来记录我们已经add进来的元素。

```
    public boolean add(E e) {
        typeCheck(e);

        long oldElements = elements;
        elements |= (1L << ((Enum<?>)e).ordinal());
        return elements != oldElements;
    }
```

每加一个元素的时候，会将elements对应的二进制的这个元素对应的索引那一位标记为1，比如我们把第一个元素加进来，就是01，再把第二个元素加进来就是02,那么，contains和size方法就会变得简单，contains只需要判断对应index位是否为1即可，size则返回二进制中1的个数。非常巧妙


### JumboEnumSet
在这里，elements变成了一个long数组。我们看下。add方法的实现，

```
    public boolean add(E e) {
        typeCheck(e);

        int eOrdinal = e.ordinal();
        int eWordNum = eOrdinal >>> 6;

        long oldElements = elements[eWordNum];
        elements[eWordNum] |= (1L << eOrdinal);
        boolean result = (elements[eWordNum] != oldElements);
        if (result)
            size++;
        return result;
    }

```

看到了吧，elements的每一个元素long，以0为例，对应Enum中，第0-64，对应的就是elements数组中第一个元素的long，也是以二进制中的1来标记。


### 总结

大概就说这么多吧，详细可以去研究代码。



