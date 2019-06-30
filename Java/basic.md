# Java基础知识

1. Vector和ArrayList的区别是什么？Vetor是如何保证线程安全的？
    它们都是AbstractList的子类，区别是Vector是线程安全的而ArrayList是线程不安全的。通过synchronized关键字修饰方法来包证线程安全的
2. HashMap和Hashtable的区别是什么？Hashtable是如何保证线程安全的？
   - Hashtable的key和value都不能为空，HashMap的key和value都可以为空。
   - 两者的父类不一样，虽然都实现了Map接口，Hashtable的父类是Dictionary而HashMap的父类是AbstractMap。
   - Hashtable是线程安全的而HashMap是线程不安全的。
   - 初始默认容量不同，Hashtable为11，HashMap为16。并且如果初始容量传入的不是2的幂，HashMap内部会进行转换成大于传入参数的最小2的幂。
   Hashtable是通过synchronized关键字修饰方法来包证线程安全的。
3. SparseArray和SparseMap的实现原理？

4. 介绍下HashMap的工作原理？
   put流程
   1. 检查table是否为null，或者size为0，如果是则执行resize
   2. 对key的hashcode做hash，在通过(n-1)&hash计算index
   3. 如果没有碰撞则直接放到数组里。
   4. 如果发生了碰撞，先以链表的形式存放
   5. 如果碰撞导致链表过长，默认大于8，就把链表转换为红黑树
   6. 如果节点已经存在就用value覆盖
   7. 如果数组即将满了，默认超过 当前容量*负载因子，就触发resize

   get流程
   1. 对key的hashcode做hash，在通过(n-1)&hash计算index
   2. 如果直接命中就返回
   3. 如果未命中就在红黑树或者链表中查找
   4. 如果都未找到返回null

5. 线程池都有哪些参数？各个参数的含义是什么
