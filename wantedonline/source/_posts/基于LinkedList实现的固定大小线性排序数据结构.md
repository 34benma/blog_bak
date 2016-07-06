---
title: 基于LinkedList实现的固定大小线性排序数据结构
date: 2016-07-06 17:52:28
tags: 
  - 并发 
  - 数据结构
categories: 基础技术
---

<h3>概要：</h3>
本文详细讲解了在Java中使用LinkedList实现一种可以设置固定大小的线性集合，该集合线程安全，需要达到业务的最优性能。

<!--more-->

<h3>1. 缘起</h3>
最近工作过程中碰到一个做周期性更新排行榜的需求。涉及的数据字段和记录条数非常多。概括如下：
1. 数据分布于后台数据库100张数据表中；
2. 每张表的数据更新非常快，每天预估数据增量在1W条左右；
3. 排行榜的数据生成来源于这100张表中，只取前面100条；

约束：

- 数据库服务目前只有一个主从，数据周期性变化，原始数据必须放MySQL持久化，不能放Redis
- 尽可能减少数据库连接获取，尽可能减少数据库查询，这些昂贵的资源必须优先保证核心业务

<h3>2. 基本思路</h3>
如果数据量比较少，我们直接放到一个List然后调用Collections.sort排序即可。但是假如我们的排行榜要求的数据量是1W或者10W，每条记录按1KB的大小计算，那么100张表的数据量就是100*10W*1KB，服务器内存早就爆表。显然这种方案是不行的。

我们需要这样一种结构，每次一张数据库表中查出前m条记录，将这m条记录插入到一个容器中，这个容器会自动将这m条记录按大小排序，从另外一张表中查出m条记录，继续放到这个容器，容器依然能保证从大到小存储前m条记录。

这就好像一个擂台，我们可以设置这个擂台大小。每次放到擂台的选手，经过比赛之后，自动从大到小排好序，如果超过擂台大小，排在后面的选手自动淘汰。新加进来的选手可以来踢馆，最终强者留下。如果擂台大小还有剩余，可以容纳更多的选手。直到达到擂台大小就进入淘汰赛。

同时，还有一个需求，我们100张表可以用100个线程一次查询出来，因此这个容器必须线程安全。同时，性能要好。

纵观JDK的集合包，似乎没有现成的数据结构可以用。那是否可以直接使用J.U.C包中线程安全的集合？理论上是可以的。通过组合比如LinkedBlockingQueue，PriorityBlockingQueue或其他线程安全的数据结构。问题是我们还要时刻保持有序，大小固定这两个要求，隐含一个性能要好的隐形需求。如果直接使用线程安全的集合，显然还要额外加锁，两把锁以上的数据结构有可能存在死锁问题。

与其苦苦死锁寻找JDK给我们的馅饼，还不如自己动手，丰衣足食。即便重复写的轮子不如大师的轮子，但是先写一个放到测试环境跑一跑，只要大流程不出问题，还是不会给公司带来损失，发现小bug后面再慢慢优化就是。

<h3>3. 初步实现</h3>
首先，这个擂台比如增删频繁，因此使用链式结构是最优选。容器的元素一开始就有序，并且后面新加的元素要有序，使用插入排序是最优选，后面新加元素要找到合适的插入位置。就必定要和容器中的元素打擂，但并没必要一一比赛，因为容器的元素已经是有序的了。这时候使用二分查找寻找待插入元素的位置便是最优解。

插入：O(1),使用链表，无需移动元素
二分查找: O(lgn)
遍历集合: O(n)

我们JDK提供的集合排序会根据数据量规模自动选择排序算法，最优性能是O(nlgn),因为集合中的元素一开始是无序的。因此相比较而言，理论上我们的集合性能上要优于直接使用JDK提供的排序方法。

元素类型不确定，比较依据也不确定，固定大小可以由使用者设置，一旦设置就不能更改。因此这里必须使用泛型，同时由用户传入一个比较器，可以使用匿名内部类。同时传入一个固定大小。容器内部每添加一个元素前都要判断，当前是否已经达到擂台大小，如果新加入的元素超出了擂台大小，则必须进入淘汰赛，将最小元素淘汰。当然，维护从大到小还是从小到大可以通过比较器的返回值决定，这里我的需求是从大到小。

综合以上的条件，我们可以动手写代码了。

<h3>4. 核心实现</h3>

```java
    public class ArenaList<E> {
    /*the arena capacity*/
    private final int capacity;
    /*need user to define，int r = compare(e1, e2),when r > 0，e1 > e2,r = 0, e1 = e2, r < 0, e1 < e2*/
    private final Comparator<E> comparator;
    
    private final LinkedList<E> values;
    /*if there is an element in list e1, that e1 equal e2, which e2 is we want to add,
     * if keepNewerElement is false,we will drop e2,else drop e1*/
    private final boolean keepNewerElement;
    /*at this moment,the element count, when eCount < size, we can add new element into list and will not remove the last element*/
    private volatile int eCount = 0;
```

容器定义的字段如上代码所示，
capacity必须在构造函数中传入，一旦初始化之后就不能更改。同时以后可能会实现序列化，虽然现在没有考虑序列化问题。
comparator是用户定义的比较器，也必须在构造时初始化。我们约定，调用比较器的compare(e1, e2)比较元素，当返回值r > 0，表示e1 > e2,当r < 0,表示e1 < e2。r = 0则表示元素相等。
values是存储元素的最终容器，因为增删频繁，因此使用链表，初始化之后就不能更改；
keepNewerElement是这样一个作用，当排在最后的两个元素en和em，如果en = em,em是待加入元素，并且擂台已满，是否用新加入的淘汰旧的。默认这个开关是false。
eCount，表示当前容器中添加了多少元素。当擂台没有满时，eCount < size，当擂台满了的时候 eCount = size。这里加了volatile修饰，是为了保证多线程下该变量的可见性。这里不深入探讨Java并发问题，因为这个问题要说清楚可以另开一篇文章来说。简单的说volatile是轻量级同步锁，可能保证变量的可见性但不能保证原子性。同时保证了一些可能引发潜在问题的重排序问题。这些涉及到Java内存模型(JMM)，不在多说。这里没有用J.U.C包下的原子变量是因为我们这里容器的增删也要加锁，为了更好的性能，就没必要CAS自旋消耗性能。

提供的两个构造函数：

```java
	public ArenaList(int capacity, Comparator<E> comparator) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("size can't be negative number");
        }
        
        this.capacity = capacity;
        this.comparator = comparator;
        this.keepNewerElement = false;
        this.values = new LinkedList<E>();
    }
```
    
默认keepNewerElement是false，我们也可以自定义为true
```java  
    public ArenaList(int capacity, Comparator<E> comparator, boolean keepNewerElement) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("size can't be negative number");
        }
        
        this.capacity = capacity;
        this.comparator = comparator;
        this.keepNewerElement = keepNewerElement;
        this.values = new LinkedList<E>();
    }
```

这个容器中，凡是涉及到容器的增删操作都要考虑线程安全问题。因此都要加锁。锁的实现也有好多种。比如粗粒度的方法锁，或JDK提供的可重入锁(ReentrantLock)。但隐含的性能要求最好时将临界区保持最小。同时代码简洁，因此这里我选用了synchronized对象锁。以values作为锁对象。

添加删除元素的部分代码如下：

```java
    /**
     * add e into list with thread-safe
     * @param e
     */
    public void add(E e) {
        if (null == e) {
            throw new NullPointerException("Null Element.");
        }
        
        synchronized (values) {
            int index = findElementIndex(e);
            if (values.size() >= capacity) {
                if (keepNewerElement && index <= values.size()) {
                    values.removeLast();
                } else if (index < values.size()){
                    //e is in the middle of this list
                    values.removeLast();
                } else {
                    //e is at the end of this list and e equals the minElement, no need to add.
                    return;
                }
            }
            values.add(index, e);
            eCount++;
        }
    }
```

删除元素:
```java    
    /**
     * remove e from list with thread-safe
     * @param e
     * @return
     */
    public boolean remove(E e) {
        if (null == e) {
            throw new NullPointerException("Null Element.");
        }
        synchronized (values) {
            if(values.remove(e)) {
                eCount--;
                return true;
            }
            return false;
        }
    }
```

寻找待插入元素元素采用的算法是二分查找，这个算法原理非常简单，但是陷阱非常多，一不小心就会陷入死循环。我还记得当年我大学毕业处女面BAT中某一家，面试官让我写个二分查找。我三下五除二洋洋洒洒写完了，完全手写，没办法测试。估计当时肯定有bug。反正最终挂了。这个算法还真不是那么简单，代码如下：
```java  
    /**
     * find the index of e should be
     * @param e
     * @return
     */
    private int findElementIndex(E e) {
        int index = 0;
        if (values.size() == 0) {
            return index;
        }
        //values.size() > 0
        E minElement = values.getLast();
        //the largest element at the index of 0
        int high = values.lastIndexOf(minElement);
        
        E maxElement = values.getFirst();
        //the least element at the index of values.size() - 1
        int low = values.indexOf(maxElement);
        
        if (compareTwoElement(minElement, e) >= 0) {
            //minElement >= e
            index = high + 1;
            return index;
        }
        
        if (compareTwoElement(maxElement, e) <= 0) {
            //maxElement <= e
            return index;
        }
        
        //only one element
        if (low == high) {
            if (capacity == 1) {
                return index;
            }
            if (compareTwoElement(minElement, e) > 0) {
                //minElement > e
                index = high + 1;
                return index;
            } else if (compareTwoElement(minElement, e) < 0) {
                //minElement < e
                return index;
            } else {
                //minElement = maxElement = e
                return index;
            }
        }
        
        //more than one element and e must at the middle of this list,use binary search to find index
        int middle = 0;
        while (low < high) {
            middle = (low + high) / 2;
            E minddleE = values.get(middle);
            if (compareTwoElement(minddleE, e) == 0) {
                //find it.
                index = middle;
                return index;
            }
            
            if (compareTwoElement(minddleE, e) > 0) {
                //middleE > e
                low = middle;
            } else {
                //middleE < e
                high = middle;
            }
            if (high - low == 1) {
                //if there only two element, e is in the middle of this list,low will never be higher or equal high
                //at this moment,should check out
                //[4,2,1],3 need to insert into it or [4,3,1],2 need to insert into it
                return low + 1;
            }
            
        }
        //at this moment,low >= high,and (middle + 1)is we should find
        index = middle + 1;
        return index;
    }
```
这里注意的是，容器中，大的元素在index为0，因此low表示大元素，high表示小元素。这点有悖于约定俗称。当只剩一个元素比较时就要特别注意了。

<h3>5. 总结</h3>
到这里，基本分析完了。我写过测试用例，尝试过使用100个线程来添加元素。目前业务上似乎没有问题。

这个容器的原理非常简单，涉及的知识点也不算难。都是大学学过的。但是真正写起来却花费了很多时间。需要想各种测试场景写各种测试代码。到现在我完全没有信心说这个容器没有任何bug。但目前基本能满足我们的业务需求。Java并发是一门非常精深非常有意思的学问。很多时候我们写出的代码功能是符合要求，但是性能上本来可以做到更好的。其中并发就是一门需要丰富的经验和钻研才能写出高并发高性能程序的学问。在这条路上，我才刚刚起步。

所有代码开源，可以自由使用。欢迎提出bug或更好的意见。特别是并发优化部分。

源码下载：[https://github.com/34benma/MyTools/blob/master/src/com/util/ArenaList.java](https://github.com/34benma/MyTools/blob/master/src/com/util/ArenaList.java "https://github.com/34benma/MyTools/blob/master/src/com/util/ArenaList.java")

<h3>6. 参考资料</h3>

- [http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/java/util/LinkedList.java#LinkedList](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/java/util/LinkedList.java#LinkedList)

