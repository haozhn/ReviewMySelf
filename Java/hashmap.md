# HashMap源码解析

## HashMap中的静态变量

- DEFAULT_INITIAL_CAPACITY：默认容量大小，值为16
- MAXIMUM_CAPACITY：
- DEFAULT_LOAD_FACTOR：
- TREEIFY_THRESHOLD：
- UNTREEIFY_THRESHOLD
- MIN_TREEIFY_CAPACITY

## HashMap的创建

HashMap有四种构造方法，代码如下

```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

第一种方法是默认构造方法，只做了负载因子的赋值工作
第二种传入初始容量的构造方法最后也调用了第三种构造方法，所以我们直接看第三种构造方法，这个方法先是对初始容量initialCapacity和负载因子loadFactor做了合法性校验，然后就是赋值了，这里有个很有趣的方法tableSizeFor，它可以接收一个参数，并返回大于这个参数的最小的2的幂，它的实现如下

```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

当时我刚看到这个方法的时候也是一脸懵逼的，如果不是有注释，我都不知道这个方法是要做什么的，不过用几个数实际验算后就发现了其中的奥秘了。首先我们要观察下2的幂都有什么规律。

```java
2               // 00000000000000000000000000000010
4               // 00000000000000000000000000000100
8               // 00000000000000000000000000001000
16              // 00000000000000000000000000010000
32              // 00000000000000000000000000100000
64              // 00000000000000000000000001000000
128             // 00000000000000000000000010000000
256             // 00000000000000000000000100000000
1073741824      // 01000000000000000000000000000000
1073741823      // 00111111111111111111111111111111
```

很容易发现它们的二进制只有一位是1，其余都是0。但是它们减1的值的二进制却是都是1，

## HashMap的put流程

## HashMap的get流程

## HashMap的Resize流程
