[TOC]



# 一、java基础

## 1、List与Set的区别

| List                                                       | Set                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| List的特征是其元素以线性方式存储，集合中可以存放重复对象。 | Set是最简单的一种集合。集合中的对象不按特定的方式排序，并且没有重复对象。 |

## 2、HashSet如何保证不重复的

1、根据存入对象的hashCode判断是否存在相同的，如果不存在，则不重复 存储。

2、如果hashCode相同，则调用equals方法判断是否相同，如果不相同，则不重复 存储。

3、如果hashCode相同，equals返回为true则重复，不存储。

## 3、HashMap是线程安全的吗，为什么不是线程安全的。

HashMap的结构是**桶链式**（在jdk1.8下如果链表的长度超过8时，会自动转换成红黑树），

![HashMap底层结构图](https://images2018.cnblogs.com/blog/884911/201803/884911-20180310143539126-1478791518.jpg)

jdk1.7版本下 HashMap底层实现是Entry数组实现。jdk1.8 HashMap底层实现是Node数组实现。

- 由于是数组的实现，那么在并发情况下，对于同一个index下标元素的处理会发生覆盖。

- 在HashMap自动扩容处理中，会讲原来的数组复制到扩容后的数组，在并发情况下，会存在数据丢失。

## 4、HashMap的扩容过程

HashMap的默认大小capacity是16，它的负载因子loadFactor是0.75，那么它的最大个数threshold = size * loadfactor 为12，那么也是说当数组的length超过12的时候，HashMap会触发扩容，首先会新建一个数组，其大小为原来的两倍，然后将原来数组的元素复制到新数组。其中复制的过程，过程详细如下。

由于我们数组的的大小一定是2的次方倍，并且扩容之后的大小为原来的两倍，所以我们设原来大小为2^N，那么扩容之后的大小为2^N+1，那么元素在put到HashMap中，它在数组的中下标是由它的key的hashCode决定的，但是由于数组的长度有限，你的HashCode是大小不能超过数组的长度，所以我们会看到在源码中有这么一行，

`i = (length - 1)& hash(key)` 这里解释一下  length是数组的长度，减一是因为下标从0开始，hash(key)就是返回key的哈希值，这样就保证i的范围在0到length-1之间，通过这一点我们可以知道 新数组怎么复制到新数组了。

- 如果第N+1位为0  则不需要任何处理，不需要进行位置调整。
- 如果第N+1位位1，则下标变成老的下标值+老的数组长度。

下面代码就是判断扩容的源码

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

其中`e.hash & oldCap` 就是判断第N+1位是否位0.可能大家看不懂是什么意思，我给大家打个比方，

比如你的e.hash也就是key的hash值位53，HashMap扩容前大小为2^4 =16 N=4， 那么扩容前这个元素在数组中的下标

index = 53 & (16-1) 换成2进制运算也就是 （补齐8位）0011 0101& 0000 1111 = 0000 0101也就是5 那么原来的下标就是5。

那么进行扩容 从16=》32  N原来是4 现在是5 ，所以我们看看按照规则0011 0101 需要判断第五位是否为1

在jdk源码中为 `if ((e.hash & oldCap) == 0)` 此时我们的e.hash为 0011 0101 oldCap为 16 也就是

0011 0101 & 0001 0000巧妙的利用 &运算符快速判断是不是为1  如果为1 则 `newTab[j + oldCap] = hiHead;` 如果为0 ` newTab[j] = loHead;`。

## 5、HashMap1.7与1.8的区别，说明1.8做了哪些优化，如何优化的

- 在1.7中 HashMap底层为Entry数组为拉链式，当hashCode值相等的时候，则在链表上添加元素,那么极端情况下需要遍历整个链表，时间复杂度为O(n)

- 在1.8中 HashMap底层为Node数组为拉链式 + 红黑树，当hashCode值相等的时候，则在链表上添加元素，当链表的长度超过8的时候，链表会转红黑树。那么极端情况下，时间复杂度为O(logn)

## 6、final finally finalize区别

- final可以用于修饰类、方法、字段。如果修饰类，代表该类不可以继承、如果修饰方法，代表该方法不可以重写、如果修饰字段，代表该字段不可以被修改。

- finally通常与try catch finally一起用，通常用于异常捕获，不管有没有异常发生，finally的代码块一会被执行。
- finalize 属于object的方法，java技术允许使用finalize（）方法在垃圾收集器将对象从内存中清除出去之前做必要的清理工作。这个方法是由垃圾收集器在确定这个对象没有被引用时对这个对象调用的。它是在object类中定义的，因此所有的类都继承了它。子类覆盖finalize（）方法以整理系统资源或者被执行其他清理工作。finalize（）方法是在垃圾收集器删除对象之前对这个对象调用的。 
