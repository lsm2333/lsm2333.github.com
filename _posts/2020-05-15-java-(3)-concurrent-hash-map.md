---
layout: post
category: Java
title: Java回望(3)-ConcurrentHashMap
tagline: by Snail
tags: 
  - Java
published: true
date: 2020-05-15 22:43:17 +08:00
---	

让我们继续Java回望系列，之前讲到HashMap这一常用的map实现，那么在线程安全方面，经常用到的Map实现就是ConcurrentHashMap了，就让我来梳理下ConcurrentHashMap中的一些知识点。
<!--more-->

## 基本概念
ConcurrentHashMap是由segment组成，可以理解为一个segment数组。每个segment通过继承ReentrantLock来实现线程安全。ConcurrencyLevel是ConcurrentHashMap中一个比较重要的设置，默认是16，意味着默认有16个segment，即最多支持16个线程并发写。可以在初始化的时候设置该值，但是一旦初始化，该参数不可更改。

ConcurrentHashMap实际上用了分段锁或者说是"锁分离"的思路，对不同的数据集分别赋予不同锁，从而降低锁的颗粒度，利用多个锁来控制多个table。

## JDK1.7版本实现
在该版本中，ConcurrentHashMap的数据结构是一个Segment数组，每个Segment元素包括了一个HashEntry数组，每个HashEntry元素又包含了一个链表。由此可见，每个HashEntry的数据结构和一个HashMap相似，都是数组+链表。

### 初始化
concurrencyLevel决定了有多少个Segment，默认为16个，其大小为2的N次方表达。每个Segment下的HashEntry长度也是按2的N次方表达，默认为2。

### put方法
会进行两次hash操作判定插入位置，同时由于Segment继承了ReentrantLock，会利用锁的特性进行上锁。具体流程如下：
1. 第一次对key进行hash，定位Segment的位置。若该Segment未初始化，则通过CAS操作进行赋值。
2. 第二次hash，找到HashEntry的位置。在插入前会做tryLock操作尝试获取锁，如果获取成功，就插入；否则会进行自旋尝试获得锁，超过一定次数则会挂起进入阻塞状态。

### get方法
同样会进行两次hash对比，第一次定位到Segment的位置，第二次定位到HashEntry的位置并遍历HashEntry上的链表，找到匹配的那个元素并返回。

### size操作
因为ConcurrentHashMap支持并发操作，那么在并发环境下，获取size时有可能和实际的size有差距（可能获取size时正在插入）。对此有两种方案：
1. 不加锁的方式获取size，最多3次，比较几次size的结果，如果一致则认为无数据插入，并返回size。
2. 在方案1不一致的情况下，给每个segment上锁，接着计算size。

## JDK1.8版本实现
1.8中放弃了segment的实现，而是和1.8中HashMap的数据结构相同，采用数组+Node链表/Node红黑树的数据结构，同时采用CAS和Synchronized来控制并发。
其中主要的几个常量列举如下：
1. MAXIMUM_CAPACITY：node数组最大容量，为2的30次方，1亿多
2. DEFAULT_CAPACITY：默认容量，为16
3. TREEIFY_THRESHOLD：链表转树阈值，为8
4. UNTREEIFY_THRESHOLD：树转链表阈值，为6
一些基本结构：
1. Node：继承于HashMap中的Entry，用于存储数据。属性包括了hash、key、val和指向下一节点的next。不允许对val进行修改，会抛出UnsupportedOperationException异常。
2. TreeNode：继承于Node，用于红黑树。属性包括了parent、left、right、prev分别指向节点在树中的周围节点。
3. TreeBin：是存储树的容器，也就是存储TreeNode的容器，提供了红黑树转换条件和锁控制的能力。
如果使用new ConcurrentHashMap()构造方法，会发现它是一个空方法。实际上是在put方法中执行map初始化动作。
### put方法
整个put方法是在做自循环，自循环内可以分为如下几个步骤：
1. 如果table没有初始化，就调用intiTable方法进行初始化。 
2. 如果没有hash冲突就直接CAS插入
3. 如果还在进行扩容操作就先进行扩容，然后不break进入下一次循环
4. 如果存在hash冲突就加锁（对链表头节点或者红黑树root节点加锁），接着按照链表或红黑树插入
5. 如果链表长度大于8，就会转成红黑树
6. 如果添加成功就调用addCount()方法统计size，并且检查是否需要扩容

### get方法
1. 计算hash值，定位到该table索引位置，如果是首节点符合就返回
2. 如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回
3. 以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null

### 扩容
- 关键方法：helpTransfer，它帮助旧的table的元素复制到新的table中。其实helpTransfer方法的目的就是调用多个工作线程一起帮助进行扩容，这样的效率就会更高。
- 关键方法：transfer，通过给每个线程分配桶区间，避免线程间的争用，通过为每个桶节点加锁，避免 putVal 方法导致数据不一致。同时，在扩容的时候，也会将链表拆成两份，这点和 HashMap 的 resize 方法类似。
- ForwardingNode：ForwardingNode是一种临时节点，在扩容进行中才会出现，它出现在旧数组上。读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容。

### CAS基本原理
CAS是一种乐观锁实现，JUC中很多组件都用到了这一概念。其基本思路是：
1. 读数据时不加锁
2. 回写数据时判断原值是否被其他线程修改
3. 若被修改则重新读取

拓展：ABA问题，CAS无法解决ABA问题。ABA问题指的是线程1将数据x从A->B->A，线程2判断x时认为数据无变化，但是实际上x已经发生了版本变化。
实现原理：
利用volatile关键字，保证内存可见性。在CAS算法里，有3个变量，分别是A、B、C。其中A是volatile修饰的。
1. 将A赋值给B，接着对B做+1操作放到C 
2. compare，比较A、B是否相等。有可能A对应的主内存数据被其他线程改了。
3. swap，若没改，则将C的值赋予A。基于volatile关键字，主内存中的数据也被更新。

## 对比1.7和1.8实现
1. 1.8降低了锁的颗粒度，锁在了节点上。而1.7锁在了segment上。
2. 1.8和1.7hashMap结构相同，不需要实现Segment结构，更加清晰。
3. 1.8引用红黑树，检索更快
4. 1.8使用了synchronized关键字，更加轻量自然

参考：
- [ConcurrentHashMap底层实现原理(JDK1.7 & 1.8)](https://www.jianshu.com/p/865c813f2726)
- [Java集合类框架学习 5.3—— ConcurrentHashMap(JDK1.8)](https://blog.csdn.net/u011392897/java/article/details/60479937)
- [并发编程——ConcurrentHashMap#transfer() 扩容逐行分析](https://www.jianshu.com/p/2829fe36a8dd)
- [《吊打面试官》系列-ConcurrentHashMap & HashTable](https://zhuanlan.zhihu.com/p/97902016)
