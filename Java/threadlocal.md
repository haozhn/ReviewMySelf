# ThreadLocal

在操作系统中我们学过进程和线程的区别，比较官方的话讲就是进程是操作系统分配资源的最小单位，而线程是操作系统调度的最小单位。线程附属于进程上，一个进程也最少要包含一个线程，进程与进程之间是相互独立的，但是一个进程的各个线程是共享进程资源的。

![threadlocal-1](assets/threadlocal-1.png)

如果我们想要给每个线程维护私有数据，则有两种实现方式

1. 进程中维护一个单独的变量表，每个线程都可以访问赋值
2. 每个线程单独维护一个变量表，线程只可以访问自己的私有变量表

第一种方式有个明显的弊端，就是多线程共享问题，多线程同时访问变量表时如何保证同步，如何尽量减少同步所带来的性能损耗都是需要考虑的问题。第二种方式由于是每个线程的私有变量表都是独立的，所以不存在同步问题。我们今天的主角ThreadLocal也正是通过第二种方式实现的

## 用法

- 创建ThreadLocal，ThreadLocal支持泛型，所以可以创建任何对象的ThreadLocal。创建ThreadLocal有两种方式，一种就是直接 **new** 一个ThreadLocal对象，另一个种则是调用静态方法withInitial

    ```java
    // 第一种方式
    ThreadLocal<String> local1 = new ThreadLocal<>();
    // 第二种方式
    ThreadLocal<String> local2 = ThreadLocal.withInitial(new Supplier<String>() {
            @Override
            public String get() {
                return "initial value2";
            }
    });
    ```

    从名字我们也可以才到，withInitial为ThreadLocal提供了一个初始值，当我们分别打印出local1和local2的值时，第一个是null，第二个则是初始值initial value2

    除了withInitial方法外，还有一个方法可以设置ThreadLocal的初始值，就是匿名内部类

    ```java
    ThreadLocal<String> local1 = new ThreadLocal<String>(){
            @Override
            protected String initialValue() {
                return "initial value1";
            }
        };
    ```

- 三个方法set，get，remove。ThreadLocal对外的方法只有这三个，从名字也容易知道这三个方法的含义

    ```java
    ThreadLocal<String> local = new ThreadLocal<>();
    System.out.println(local.get());  // null
    local.set("111");
    System.out.println(local.get());  // 111
    local.set("222");
    System.out.println(local.get());  // 222
    local.remove();
    System.out.println(local.get());  // null
    ```

ThreadLocal有哪些使用场景呢，在ThreadLocal源码注释里有一个简单的例子

```java
public class ThreadId {
    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
            new ThreadLocal<Integer>() {
                @Override
                protected Integer initialValue() {
                    return nextId.getAndIncrement();
                }
            };

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(ThreadId.get());
                }
            }).start();
        }
    }
}
```

ThreadId可以为每个线程分配一个唯一的id。ThreadLocal在Android中也有非常重要的应用，Android的Handler机制中的Looper就是保存的ThreadLocal中。

## ThreadLocal原理

ThreadLocal对外的方法只有三个：set，get和remove。所以我们可以从这三个方法入手研究下其实现原理。

### set方法

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

- 获取当前的运行的线程
- 获取当前线程的内部变量threadLocals
- 如果当前线程的threadLocals为空，就创建一个新的ThreadLocalMap对象
- 如果当前线程的threadLocals不为空，就调用threadLocals的set方法

在Thread类中有一个threadLocals变量，初始值为null，在第一次调用ThreadLocal的set或者get方法时，会创建一个ThreadLocalMap对象并赋值给threadLocals，ThreadLocalMap我们可以把它暂时当做HashMap，可以保存key-value，后面会细讲ThreadLocalMap的实现。

### get方法

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

- 获取当前的运行的线程
- 获取当前线程的内部变量threadLocals
- 如果threadLocals不为空，就调用map的getEntry方法，如果entry不为空，说明get方法命中，直接返回entry的value值
- 如果threadLocals为空，则会调用setInitialValue方法
- setInitialValue方法和set非常类似，setInitialValue基本和set(initialValue())的效果是一样的。

### remove方法

```java
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

这个方法很简单，就是调用了ThreadLocalMap的remove方法而已。

### ThreadLocalMap实现原理

ThreadLocalMap是一种自定义的散列表，与我们常用的HashMap相似但是也有些不同。ThreadLocalMap并没有实现Map接口，而且它的key只能是ThreadLocal对象
