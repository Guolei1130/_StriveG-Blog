### 前言

for循环和foreach是我们开发中使用频率超高的代码端。一些很小的习惯和细节能帮助我们写出更好的代码。因此，我们要了解下for和foreach是咋回事。


### 写两段测试代码

```
public class Test {

    public static void main(String[] args) {
        List<String> strings = new ArrayList<>();
        for (int i = 0; i < 1000000; i++) {
            strings.add(String.valueOf(i));
        }

        long start = System.currentTimeMillis();
        testFor(strings);
        System.err.println(System.currentTimeMillis() - start);

        start = System.currentTimeMillis();
        testForEach(strings);
        System.err.println(System.currentTimeMillis() - start);

        start = System.currentTimeMillis();
        testInter(strings);
        System.err.println(System.currentTimeMillis() - start);

    }

    public static void testFor(List<String> strings) {
        for (int i = 0; i < strings.size(); i++) {
//            System.err.println(strings.get(i));
        }
    }

    public static void testForEach(List<String> strings) {
        for (String s : strings) {
//            System.err.println(s);
        }
    }

    public static void testInter(List<String> strings) {
        Iterator<String> stringIterator = strings.iterator();
        while (stringIterator.hasNext()) {
//            System.err.println(stringIterator.next());
            stringIterator.next();
        }
    }

}
```

在我们的机器上，运行时间分别为3 16 6，可以看出，for循环的性能是要优与foreach的，即使我们这里没对for循环进行小优化，当我们把for循环中的strings.size提到外边之后，性能也会提升一点点。当然，循环体中的代码运行时间越长，优化的提升率越低。

### 编译一下，查看字节码

```

  public static void testFor(java.util.List<java.lang.String>);
    Code:
       0: iconst_0
       1: istore_1
       2: iload_1
       3: aload_0
       4: invokeinterface #13,  1           // InterfaceMethod java/util/List.size:()I
       9: if_icmpge     18
      12: iinc          1, 1
      15: goto          2
      18: return

  public static void testForEach(java.util.List<java.lang.String>);
    Code:
       0: aload_0
       1: invokeinterface #14,  1           // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
       6: astore_1
       7: aload_1
       8: invokeinterface #15,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
      13: ifeq          29
      16: aload_1
      17: invokeinterface #16,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
      22: checkcast     #17                 // class java/lang/String
      25: astore_2
      26: goto          7
      29: return
```


可以很清晰的看到，指令数是有差别的的看到。，除去istore iload等指令的差异外最重要的一点是，foreach中比for多调用一个方法，这个方法内部耗时的。因此，for和foreach中，有比较大的性能差别。

