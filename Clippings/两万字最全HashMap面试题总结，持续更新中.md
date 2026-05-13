---
title: "两万字最全HashMap面试题总结，持续更新中"
source: "https://zhuanlan.zhihu.com/p/506233341"
author:
  - "[[互联网科技小于哥互联网科技|技术开发|毕设|内推|简历指导|毕设|圆桌嘉宾]]"
published:
created: 2026-05-13
description: "我历经两个工作日晚上总结了50道关于hashmap的面试题，我相信，下面将会是Hashmap最全的面试总结！ 1.HashMap的底层数据结构？HashMap底层实现数据结构为数组+链表的形式，JDK8及其以后的版本中使用了数组+链表+红…"
tags:
  - "clippings"
---
[收录于 · 终端研发部](https://www.zhihu.com/column/c_1171818504071741440)

16 人赞同了该文章

我历经两个工作日晚上总结了50道关于hashmap的面试题，我相信，下面将会是Hashmap最全的面试总结！

## 1.HashMap的底层数据结构？

- HashMap底层实现数据结构为数组+链表的形式， [JDK8](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=JDK8&zhida_source=entity) 及其以后的版本中使用了数组+链表+ [红黑树](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=%E7%BA%A2%E9%BB%91%E6%A0%91&zhida_source=entity) 实现，解决了链表太长导致的查询速度变慢的问题。
- 简单来说，HashMap由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决 [哈希冲突](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=%E5%93%88%E5%B8%8C%E5%86%B2%E7%AA%81&zhida_source=entity) 而存在的。HashMap通过key的HashCode经过扰动函数处理过后得到Hash值，然后通过位运算判断当前元素存放的位置，如果当前位置存在元素的话，就判断该元素与要存入的元素的hash值以及key是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。当Map中的元素总数超过Entry数组的0.75时，触发扩容操作，为了减少链表长度，元素分配更均匀。

## 2.hash的计算规则？

![](https://pica.zhimg.com/v2-9fb1b401f781a631e3e5de34893d916e_1440w.jpg)

## 2.1为什么这里把key的hashcode取出来，然后把它右移16位，然后取异或？

因为int是4个字节，也就是32位，大概是有40亿的空间，如果哈希函数运用的比较松散，一般是很难出现哈希碰撞的。但是现实中一个长度为40亿的数组内存是放不下的并且HashMap在扩容前的数组的默认初始值为16，因此直接拿Hashcode值来用是不现实的。因此需要做一些运算。我们右移16位也即是把高位的数据右移到低位的16位，然后与自己做异或，那就是把高位和低位的数据进行混合，以此来加大低位的随机性，同时混合后的低位掺杂了高位的特征，这样高位的信息也被变相保存了下来。这么做主要是从速度，功效和质量来考虑的。

## 3.默认初始化大小是多少？为啥是这么多？为啥大小都是2的幂？

hash运算的过程其实就是对目标元素的Key进行hashcode，再对Map的容量进行取模，而JDK 的工程师为了提升取模的效率，使用位运算代替了取模运算，这就要求Map的容量一定得是2的幂。       HashMap的容量为什么是2的n次幂，和这个（n - 1) & hash的计算方法有着千丝万缕的关系，符号&是按位与的计算，这是位运算，计算机能直接运算，特别高效，按位与&的计算方法是，只有当对应位置的数据都为1时，运算结果也为1，当HashMap的容量是2的n次幂时，(n-1)的2进制也就是1111111\*\*\*111这样形式的，这样与添加元素的hash值进行位运算时，能够（充分的散列），使得添加的元素均匀分布在HashMap的每个位置上，减少hash碰撞。

## 4.HashMap的主要参数都有哪些？

- DEFAULT\_INITIAL\_CAPACITY：默认的初始化容量，1<<4位运算的结果是16，也就是默认的初始化容量为16。当然如果对要存储的数据有一个估计值，最好在初始化的时候显示的指定容量大小，减少扩容时的数据搬移等带来的效率消耗。同时，容量大小需要是2的整数倍。
- MAXIMUM\_CAPACITY：容量的最大值，1 << 30位，2的30次幂。 DEFAULT\_LOAD\_FACTOR：默认的加载因子，设计者认为这个数值是基于时间和空间消耗上最好的数值。这个值和容量的乘积是一个很重要的数值，也就是阈值，当达到这个值时候会产生扩容，扩容的大小大约为原来的二倍。
- TREEIFY\_THRESHOLD：因为jdk8以后，HashMap底层的存储结构改为了数组+链表+红黑树的存储结构（之前是数组+链表），刚开始存储元素产生碰撞时会在碰撞的数组后面挂上一个链表，当链表长度大于这个参数时，链表就可能会转化为红黑树，为什么是可能后面还有一个参数，需要他们两个都满足的时候才会转化。
- UNTREEIFY\_THRESHOLD：介绍上面的参数时，我们知道当长度过大时可能会产生从链表到红黑树的转化，但是，元素不仅仅只能添加还可以删除，或者另一种情况，扩容后该数组槽位置上的元素数据不是很多了，还使用红黑树的结构就会很浪费，所以这时就可以把红黑树结构变回链表结构，什么时候变，就是元素数量等于这个值也就是6的时候变回来（元素数量指的是一个数组槽内的数量，不是HashMap中所有元素的数量）。
- MIN\_TREEIFY\_CAPACITY：链表树化的一个标准，前面说过当数组槽内的元素数量大于8时可能会转化为红黑树，之所以说是可能就是因为这个值，当数组的长度小于这个值的时候，会先去进行扩容，扩容之后就有很大的可能让数组槽内的数据可以更分散一些了，也就不用转化数组槽后的存储结构了。当然，长度大于这个值并且槽内数据大于8时，那就转化为红黑树吧。

## 5.哈希冲突及解决方法

如果两个不同对象的hashCode相同，这种现象称为hash冲突。有以下的方式可以解决哈希冲突：

1. [开放定址法](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=%E5%BC%80%E6%94%BE%E5%AE%9A%E5%9D%80%E6%B3%95&zhida_source=entity) 开放定址法就是一旦发生了冲突，就去寻找下一个空的散列地址，只要散列表足够大，空的散列地址总能找到，并将记录存入。
2. [链地址法](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=%E9%93%BE%E5%9C%B0%E5%9D%80%E6%B3%95&zhida_source=entity) 链地址法 将哈希表的每个单元作为链表的头结点，所有哈希地址为i的元素构成一个同义词链表。即发生冲突时就把该关键字链在以该单元为头结点的链表的尾部。
3. [再哈希法](https://zhida.zhihu.com/search?content_id=200628330&content_type=Article&match_order=1&q=%E5%86%8D%E5%93%88%E5%B8%8C%E6%B3%95&zhida_source=entity) 当哈希地址发生冲突用其他的函数计算另一个哈希函数地址，直到冲突不再产生为止。
4. 建立公共溢出区 将哈希表分为基本表和溢出表两部分，发生冲突的元素都放入溢出表中。

## 6.HashMap如何有效减少碰撞？

1. 扰动函数：促使元素位置分布均匀，减少碰撞几率
2. 使用final对象，并采用合适的equals()和hashCode()方法

## 7.HashMap可以实现同步吗？

HashMap可以通过下面的语句进行同步：      Map m = Collections.synchronizeMap(hashMap);

## 8.为啥我们重写equals方法的时候需要重写hashCode方法呢？

hashmap中value的查找是通过 key 的 hashcode 来查找，所以对自己的对象必须重写 hashcode 方法通过 hashcode 找到对象地址后会用 equals 比较你传入的对象和 hashmap 中的 key 对象是否相同,因此还要重写 equals。

## 9.HashMap什么时候进行扩容？它是怎么扩容的呢？

HashMap进行扩容取决于以下两个元素：

- Capacity：HashMap当前长度。
- LoadFactor：负载因子，默认值0.75f。      当Map中的元素个数（包括数组，链表和红黑树中）超过了16\*0.75=12之后开始扩容。      具体怎么进行扩容呢？将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。这个过程叫作rehashing ，因为它将会调用hash方法找到新的bucket位置。

## 10.JDK1.7扩容的时候为什么要重新Hash呢，为什么不直接复制过去？

是因为长度扩大以后，Hash的规则也随之改变。比如原来长度（Length）是8，你位运算出来的值是2 ，新的长度是16你位运算出来的值明显不一样了。

## 11.和Hashtable的区别是什么？

HashMap和Hashtable都实现了Map接口，但决定用哪一个之前先要弄清楚它们之间的分别。主要的区别有： 线程安全性，同步(synchronization)，以及速度。      HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。      HashMap是非synchronized，而Hashtable是synchronized，这意味着Hashtable是线程安全的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。      另一个区别是HashMap的迭代器(Iterator)是fail-fast迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。这条同样也是Enumeration和Iterator的区别。      由于Hashtable是线程安全的也是synchronized，所以在单线程环境下它比HashMap要慢。如果你不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。      HashMap不能保证随着时间的推移Map中的元素次序是不变的。 术语tips：

1. sychronized意味着在一次仅有一个线程能够更改Hashtable。就是说任何线程要更新Hashtable时要首先获得同步锁，其它线程要等到同步锁被释放之后才能再次获得同步锁更新Hashtable。
2. Fail-safe和iterator迭代器相关。如果某个集合对象创建了Iterator或者ListIterator，然后其它的线程试图“结构上”更改集合对象，将会抛出ConcurrentModificationException异常。但其它线程可以通过set()方法更改集合对象是允许的，因为这并没有从“结构上”更改集合。但是假如已经从结构上进行了更改，再调用set()方法，将会抛出IllegalArgumentException异常。
3. 结构上的更改指的是删除或者插入一个元素，这样会影响到map的结构。

## 12.什么是Java集合中的快速失败（fast-fail）机制?

快速失败是Java集合的一种错误检测机制，当多个线程对集合进行结构上的改变的操作时，有可能会产生fail-fast。      举个例子：假设存在两个线程（线程1、线程2），线程1通过Iterator在遍历集合A中的元素，在某个时候线程2修改了集合A的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就可能会抛出 ConcurrentModificationException异常，从而产生fast-fail快速失败。

### 12.1那么快速失败机制底层是怎么实现的呢？

迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个 modCount 变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedModCount值，是的话就返回遍历；否则抛出异常，终止遍历。      看异常ConcurrentModificationException，JDK中是这么介绍该异常的：当检测到一个并发的修改，就可能会抛出该异常，一些迭代器的实现会抛出该异常，以便可以快速失败。但是你不可以为了便捷而依赖该异常，而应该仅仅作为一个程序的侦测。

## 13.HashTable一定是线程安全吗？它会有快速失败的时候吗？

Hashtable线程安全是由于其内部实现在put和remove等方法上使用synchronized进行了同步，所以对单个方法的使用是线程安全的。但是对多个方法进行复合操作时，线程安全性无法保证。 比如一个线程在进行get操作，一个线程在进行remove操作，往往会导致下标越界等异常。      Hashtable也会在迭代的时候抛出ConcurrentModificationException，可能发生快速失败。

## 14.为什么String, Interger这样的wrapper类适合作为键？

String, Interger这样的wrapper类作为HashMap的键是再适合不过了，而且String最为常用。      因为String是不可变的，也是final的，而且已经重写了equals()和hashCode()方法了。其他的wrapper类也有这个特点。不可变性是必要的，因为为了要计算hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的hashcode的话，那么就不能从HashMap中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个field声明成final就能保证hashCode是不变的，那么请这么做吧。因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的hashcode的话，那么碰撞的几率就会小些，这样就能提高HashMap的性能。

14.1我们可以使用自定义的对象作为键吗？

这是前一个问题的延伸。当然你可能使用任何对象作为键，只要它遵守了equals()和hashCode()方法的定义规则，并且当对象插入到Map中之后将不会再改变了。如果这个自定义对象时不可变的，那么它已经满足了作为键的条件，因为当它创建之后就已经不能改变了。

## 15.HashMap的工作原理?

```
HashMap
```

底层是hash数组和单向链表实现,数组中的每个元素都是链表,由Node内部类(实现

```
Map.Entry<K,V>
```

接口)实现,

```
HashMap
```

通过

```
put&get
```

方法存储和获取。

**存储对象时,将**

```
K/V
```

**键值传给**

```
put()
```

**方法:**

- ①、调用hash(K)方法计算K的hash值,然后结合数组长度,计算得数组下标;
- ②、调整数组大小(当容器中的元素个数大于capacity\*loadfactor时,容器会进行扩容resize为2n);
- ③
- i.如果K的 hash 值在HashMap 中不存在,则执行插入,若存在,则发生碰撞;
- ii.如果K的 hash 值在HashMap 中存在,且它们两者equals 返回true,则更新键值对;
- iii.如果K的 hash 值在HashMap 中存在,且它们两者equals 返回false,则插入链表的尾部(尾插法)或者红黑树中(树的添加方式)。

> ( JDK1.7 之前使用头插法、JDK1.8 使用尾插法) (注意:当碰撞导致链表大于TREEIFY\_THRESHOLD=8 时,就把链表转换成红黑树)

**获取对象时,将**

```
K
```

**传给**

```
get()
```

**方法:**

- ①、调用hash(K)方法(计算K的hash值)从而获取该键值所在链表的数组下标;
- ②、顺序遍历链表,equals()方法查找相同Node链表中K值对应的V值。
```
hashCode
```

是定位的,存储位置;

```
equals
```

是定性的,比较两者是否相等。

## 16.当两个对象的hashCode相同会发生什么?

因为

```
hashCode
```

相同,不一定就是相等的(

```
equals
```

方法比较),所以两个对象所在数组的下标相同,"碰撞"就此发生。又因为

```
HashMap
```

使用链表存储对象,这个

```
Node
```

会存储到链表中。

## 17.你知道hash的实现吗?为什么要这样实现?

```
JDK1.8
```

中,是通过

```
hashCode()
```

的高16位异或低16位实现的:

```
(h=k.hashCode())^(h>>>16)
```

,主要是从速度,功效和质量来考虑的,减少系统的开销,也不会造成因为高位没有参与下标的计算,从而引起的碰撞。

![](https://pic3.zhimg.com/v2-6356db31af6fe3db61445fe4f6165354_1440w.jpg)

计算过程如下所示：

说明：

- key.hashCode()；返回散列值也就是hashcode，假设随便生成的一个值。
- n表示数组初始化的长度是16。
- &（按位与运算）：运算规则：相同的二进制数位上，都是1的时候，结果为1，否则为零。
- ^（按位异或运算）：运算规则：相同的二进制数位上，数字相同，结果为0，不同为1。
![](https://pic2.zhimg.com/v2-b8abb909358659e3924a2433faf3bd7f_1440w.jpg)

简单来说就是：

**高16bit不变，低16bit和高16bit做了一个异或（得到的hashCode转化为32位二进制，前16位和后16位低16bit和高16bit做了一个异或）**

问题：为什么要这样操作呢？

如果当n即数组长度很小，假设是16的话，那么n - 1即为1111 ，这样的值和hashCode直接做按位与操作，实际上只使用了哈希值的后4位。如果当哈希值的高位变化很大，低位变化很小，这样就很容易造成哈希冲突了，所以这里把高低位都利用起来，从而解决了这个问题。

![](https://picx.zhimg.com/v2-db52370ce479b8d5dc4247feb99216c5_1440w.jpg)

## 18.为什么要用异或运算符?

保证了对象的

```
hashCode
```

的32位值只要有一位发生改变,整个

```
hash()
```

返回值就会改变。尽可能的减少碰撞。

## 19.HashMap的table的容量如何确定?loadFactor是什么?该容量如何变化?这种变化会带来什么问题?

- ①、 table 数组大小是由capacity 这个参数确定的,默认是16,也可以构造时传入,最大限制是1<<30;
- ②、 loadFactor 是装载因子,主要目的是用来确认table 数组是否需要动态扩展,默认值是0.75,比如table 数组大小为16,装载因子为0.75时,threshold 就是12,当table 的实际大小超过12时,table 就需要动态扩容;
- ③、扩容时,调用 resize() 方法,将table 长度变为原来的两倍(注意是table长度,而不是threshold )
- ④、如果数据很大的情况下,扩展时将会带来性能的损失,在性能要求很高的地方,这种损失很可能很致命。

## 20.HashMap中put方法的过程?

- 调用哈希函数获取 Key 对应的hash 值,再计算其数组下标;
- 如果没有出现哈希冲突,则直接放入数组;如果出现哈希冲突,则以链表的方式放在链表后面;
- 如果链表长度超过阀值( TREEIFYTHRESHOLD==8 ),就把链表转成红黑树,链表长度低于6,就把红黑树转回链表;
- 如果结点的 key 已经存在,则替换其value 即可;
- 如果集合中的键值对大于12,调用 resize 方法进行数组扩容。
![](https://picx.zhimg.com/v2-a56c94266731f981c42d8bdc9bcbe757_1440w.jpg)

## 21.数组扩容的过程?

创建一个新的数组,其容量为旧数组的两倍,并重新计算旧数组中结点的存储位置。结点在新数组中的位置只有两种,原下标位置或原下标+旧数组的大小。

什么时候才需要扩容

当HashMap中的元素个数超过数组大小(数组长度)\*loadFactor(负载因子)时，就会进行数组扩容，loadFactor的默认值(DEFAULT\_LOAD\_FACTOR)是0.75,这是一个折中的取值。也就是说，默认情况下，数组大小为16，那么当HashMap中的元素个数超过16×0.75=12(这个值就是阈值或者边界值threshold值)的时候，就把数组的大小扩展为2×16=32，即扩大一倍，然后重新计算每个元素在数组中的位置，而这是一个非常耗性能的操作，所以如果我们已经预知HashMap中元素的个数，那么预知元素的个数能够有效的提高HashMap的性能。

补充：

当HashMap中的其中一个链表的对象个数如果达到了8个，此时如果数组长度没有达到64，那么HashMap会先扩容解决，如果已经达到了64，那么这个链表会变成红黑树，结点类型由Node变成TreeNode类型。当然，如果映射关系被移除后，下次执行resize方法时判断树的结点个数低于6，也会再把树转换为链表。

HashMap的扩容是什么

进行扩容，会伴随着一次重新hash分配，并且会遍历hash表中所有的元素，是非常耗时的。在编写程序中，要尽量避免resize。

HashMap在进行扩容时，使用的rehash方式非常巧妙，因为每次扩容都是翻倍，与原来计算的 (n-1)&hash的结果相比，只是多了一个bit位，所以结点要么就在原来的位置，要么就被分配到"原位置+旧容量"这个位置。

![](https://pica.zhimg.com/v2-e5c73277da9332ad36805a3551a7988e_1440w.jpg)

![](https://pica.zhimg.com/v2-6496f3d0dca3d2d638901485a004991e_1440w.jpg)

![](https://pic2.zhimg.com/v2-d659046a1033a12a9740c5fd1da9cac9_1440w.jpg)

## 22.拉链法导致的链表过深问题为什么不用二叉查找树代替,而选择红黑树?为什么不一直使用红黑树?

之所以选择红黑树是为了解决二叉查找树的缺陷,二叉查找树在特殊情况下会变成一条线性结构(这就跟原来使用链表结构一样了,造成很深的问题),遍历查找会非常慢。而红黑树在插入新数据后可能需要通过左旋,右旋、变色这些操作来保持平衡,引入红黑树就是为了查找数据快,解决链表查询深度的问题,我们知道红黑树属于平衡二叉树,但是为了保持"平衡"是需要付出代价的,但是该代价所损耗的资源要比遍历线性链表要少,所以当长度大于8的时候,会使用红黑树,如果链表长度很短的话,根本不需要引入红黑树,引入反而会慢。

![](https://picx.zhimg.com/v2-e82f3e2bfc9b1a7d598849787772fbbb_1440w.jpg)

## 23.说说你对红黑树的见解?

- 1、每个节点非红即黑
- 2、根节点总是黑色的
- 3、如果节点是红色的,则它的子节点必须是黑色的(反之不一定)
- 4、每个叶子节点都是黑色的空节点(NIL节点)
- 5、从根节点到叶节点或空子节点的每条路径,必须包含相同数目的黑色节点(即相同的黑色高度)

## 24.jdk8中对HashMap做了哪些改变?

- 在 java1.8 中,如果链表的长度超过了8,那么链表将转换为红黑树。(桶的数量必须大于64,小于64的时候只会扩容)
- 发生 hash 碰撞时,java1.7 会在链表的头部插入,而java1.8 会在链表的尾部插入
- 在 java1.8 中,Entry 被Node 替代(换了一个马甲)。

## 25.HashMap,LinkedHashMap,TreeMap有什么区别?

- HashMap 参考其他问题;
- LinkedHashMap 保存了记录的插入顺序,在用Iterator 遍历时,先取到的记录肯定是先插入的;遍历比HashMap 慢;
- TreeMap 实现SortMap 接口,能够把它保存的记录根据键排序(默认按键值升序排序,也可以指定排序的比较器)

## 26.HashMap&TreeMap&LinkedHashMap使用场景?

一般情况下,使用最多的是

```
HashMap
```

**HashMap:** 在Map中插入、删除和定位元素时;

**TreeMap:** 在需要按自然顺序或自定义顺序遍历键的情况下;

**LinkedHashMap:** 在需要输出的顺序和输入的顺序相同的情况下。

## 27.HashMap和HashTable有什么区别?

- ①、 HashMap 是线程不安全的,HashTable 是线程安全的;
- ②、由于线程安全,所以 HashTable 的效率比不上HashMap;
- ③、 HashMap 最多只允许一条记录的键为null,允许多条记录的值为null,而HashTable 不允许;
- ④、 HashMap 默认初始化数组的大小为16,HashTable 为11,前者扩容时,扩大两倍,后者扩大两倍+1;
- ⑤、 HashMap 需要重新计算hash值,而HashTable 直接使用对象的hashCode;

## 28.HashMap 的底层数组长度为何总是2的n次方

HashMap根据用户传入的初始化容量，利用无符号右移和按位或运算等方式计算出第一个大于该数的2的幂。

- 使数据分布均匀，减少碰撞
- 当length为2的n次方时，h&(length - 1) 就相当于对length取模，而且在速度、效率上比直接取模要快得多

## 29.jdk1.8中做了哪些优化优化？

1. 数组+链表改成了数组+链表或红黑树
2. 链表的插入方式从头插法改成了尾插法
3. 扩容的时候1.7需要对原数组中的元素进行重新hash定位在新数组的位置，1.8采用更简单的判断逻辑，位置不变或索引+旧容量大小；
4. 在插入时，1.7先判断是否需要扩容，再插入，1.8先进行插入，插入完成再判断是否需要扩容；

## 30.HashMap线程安全方面会出现什么问题

- 在jdk1.7中，在多线程环境下，扩容时会造成环形链或数据丢失。
- 在jdk1.8中，在多线程环境下，会发生数据覆盖的情况

## 31.为什么HashMap的底层数组长度为何总是2的n次方

这里我觉得可以用逆向思维来解释这个问题，我们计算桶的位置完全可以使用h % length，如果这个length是随便设定值的话当然也可以，但是如果你对它进行研究，设计一个合理的值得话，那么将对HashMap的性能发生翻天覆地的变化。

没错，JDK源码作者就发现了，那就是当length为2的N次方的时候，那么，为什么这么说呢？

**第一：当length为2的N次方的时候，h & (length-1) = h % length** 为什么&效率更高呢？因为位运算直接对内存数据进行操作，不需要转成十进制，所以位运算要比取模运算的效率更高

**第二：当length为2的N次方的时候，数据分布均匀，减少冲突**

## 32.那么为什么默认是16呢？怎么不是4？不是8？

关于这个默认容量的选择，JDK并没有给出官方解释，那么这应该就是个经验值，既然一定要设置一个默认的2^n 作为初始值，那么就需要在效率和内存使用上做一个权衡。这个值既不能太小，也不能太大。

太小了就有可能频繁发生扩容，影响效率。太大了又浪费空间，不划算。

所以，16就作为一个经验值被采用了。

**在JDK1.8的 236 行有1<<4就是16** ，为啥用位运算呢？直接写16不好么？

![](https://picx.zhimg.com/v2-bf9de2ecfa4316908b39ba9ea4ca4c75_1440w.jpg)

我们在创建HashMap的时候，阿里巴巴规范插件会提醒我们最好赋初值，而且最好是2的幂。

![](https://pic3.zhimg.com/v2-f21e5a21d1a6303ec048c90c1b253552_1440w.jpg)

这样是为了位运算的方便， **位与运算比算数计算的效率高了很多** ，之所以选择16，是为了服务将Key映射到index的算法。

我前面说了所有的key我们都会拿到他的hash，但是我们怎么尽可能的得到一个均匀分布的hash呢？

是的我们通过Key的HashCode值去做位运算。

我们再看下index的计算公式：index = HashCode（Key） & （Length- 1） 15的的二进制是1111，那10111011000010110100 &1111 十进制就是4

之所以用位与运算效果与取模一样，性能也提高了不少！

那为啥用16不用别的呢？

因为在使用不是2的幂的数字的时候，Length-1的值是所有二进制位全为1，这种情况下，index的结果等同于HashCode后几位的值。

只要输入的HashCode本身分布均匀，Hash算法的结果就是均匀的。这是为了实现均匀分布。

当length为奇数时，length-1为偶数，而偶数二进制的最后一位永远为0，那么与其进行 & 运算，得到的二进制数最后一位永远为0，那么结果一定是偶数，那么就会导致下标为奇数的桶永远不会放置数据，这就不符合我们均匀放置，减少冲突的要求了。

那么可能钻牛角尖的同学还会问，那length是偶数不就行了么，为什么一定要是2的N次方，这不就又回到第一点原因了么？（ **当length为2的N次方的时候，h & (length-1) = h % length** ）JDK 的工程师把各种位运算运用到了极致，想尽各种办法优化效率。

## 33.HashMap的不安全体现在哪里？

hashMap是 **线程不安全** 的，其主要体现：

> 1.在jdk1.7中，在多线程环境下，扩容时会造成环形链或数据丢失。 2.在jdk1.8中，在多线程环境下，会发生数据覆盖的情况。

现在我们要在容量为2的容器里面 **用不同线程** 插入A，B，C，假如我们在resize之前打个短点，那意味着数据都插入了但是还没resize那扩容前可能是这样的。

我们可以看到链表的指向A->B->C

**Tip：A的下一个指针是指向B的**

![](https://pica.zhimg.com/v2-b9b59a2c52d22068b413ca50ddb41a64_1440w.jpg)

因为resize的赋值方式，也就是使用了 **单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置** ，在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。

就可能出现下面的情况，大家发现问题没有？

B的下一个指针指向了A

一旦几个线程都调整完成，就可能出现环形链表

![](https://picx.zhimg.com/v2-ee384d9ad13be1269e3b6ea413f691e5_1440w.jpg)

如果这个时候去取值，悲剧就出现了——Infinite Loop。

jdk1.8中HashMap中put操作的主函数，如果没有hash碰撞则会直接插入元素。如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第6行代码中。假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给 **覆盖** ，发生线程不安全。

## 34.为什么JDK1.8使用红黑树？

比如某些人通过找到你的hash碰撞值，来让你的HashMap不断地产生碰撞，那么相同key位置的链表就会不断增长，当你需要对这个HashMap的相应位置进行查询的时候，就会去循环遍历这个超级大的链表，性能及其地下。java8使用红黑树来替代超过8个节点数的链表后，查询方式性能得到了很好的提升，从原来的是O(n)到O(logn)。

## 35.1.8中的扩容为什么逻辑判断更简单

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![](https://pic2.zhimg.com/v2-a399fef34ff450516530ffa6a6d2b183_1440w.jpg)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

![](https://pica.zhimg.com/v2-40e99c9e1380a17c2817192e9e02897e_1440w.jpg)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。

## 36.HashMap中容量的初始化

当我们使用HashMap(int initialCapacity)来初始化容量的时候，jdk会默认帮我们计算一个相对合理的值当做初始容量。那么，是不是我们只需要把已知的HashMap中即将存放的元素个数直接传给initialCapacity就可以了呢？

关于这个值的设置，在《阿里巴巴Java开发手册》有以下建议：

![](https://picx.zhimg.com/v2-563f21447136fbb332f8939a9c915197_1440w.jpg)

也就是说，如果我们设置的默认值是7，经过Jdk处理之后，会被设置成8，但是，这个HashMap在元素个数达到 8\*0.75 = 6的时候就会进行一次扩容，这明显是我们不希望见到的。我们应该尽量减少扩容。原因也已经分析过。

![](https://pic2.zhimg.com/v2-003bd50e7686b703372dd0514cf60349_1440w.jpg)

![](https://picx.zhimg.com/v2-64779db85db10daa07d5fc9dca538597_1440w.jpg)

![](https://pica.zhimg.com/v2-62527d63f8abe29e3d346dd706cc939a_1440w.jpg)

如果我们通过initialCapacity/ 0.75F + 1.0F计算，7/0.75 + 1 = 10,10经过Jdk处理之后，会被设置成16，这就大大的减少了扩容的几率。

当HashMap内部维护的哈希表的容量达到75%时（默认情况下），会触发rehash，而rehash的过程是比较耗费时间的。所以初始化容量要设置成initialCapacity/0.75 + 1的话，可以有效的减少冲突也可以减小误差。

所以，我可以认为，当我们明确知道HashMap中元素的个数的时候，把默认容量设置成initialCapacity/ 0.75F + 1.0F是一个在性能上相对好的选择，但是，同时也会牺牲些内存。

我们想要在代码中创建一个HashMap的时候，如果我们已知这个Map中即将存放的元素个数，给HashMap设置初始容量可以在一定程度上提升效率。

但是，JDK并不会直接拿用户传进来的数字当做默认容量，而是会进行一番运算，最终得到一个2的幂。原因也已经分析过。

但是，为了最大程度的避免扩容带来的性能消耗，我们建议可以把默认容量的数字设置成initialCapacity/ 0.75F + 1.0F。

## 37.HashMap的put方法的具体流程？

当我们put的时候，首先计算

```
key
```

的

```
hash
```

值，这里调用了

```
hash
```

方法，

```
hash
```

方法实际是让

```
key.hashCode()
```

与

```
key.hashCode()>>>16
```

进行异或操作，高16bit补0，一个数和0异或不变，所以 hash 函数大概的作用就是： **高16bit不变，低16bit和高16bit做了一个异或，目的是减少碰撞** 。按照函数注释，因为bucket数组大小是2的幂，计算下标

```
index = (table.length - 1) & hash
```

，如果不做 hash 处理，相当于散列生效的只有几个低 bit 位，为了减少散列的碰撞，设计者综合考虑了速度、作用、质量之后，使用高16bit和低16bit异或来简单处理减少碰撞，而且JDK8中用了复杂度 O（logn）的树结构来提升碰撞下的性能。

## 38.HashMap是怎么解决哈希冲突的？

答：在解决这个问题之前，我们首先需要知道 **什么是哈希冲突** ，而在了解哈希冲突之前我们还要知道 **什么是哈希** 才行；

什么是哈希？

**Hash，一般翻译为“散列”，也有直接音译为“哈希”的，这就是把任意长度的输入通过散列算法，变换成固定长度的输出，该输出就是散列值（哈希值）** ；这种转换是一种压缩映射，也就是，散列值的空间通常远小于输入的空间，不同的输入可能会散列成相同的输出，所以不可能从散列值来唯一的确定输入值。 **简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数** 。

所有散列函数都有如下一个基本特性\*\*：根据同一散列函数计算出的散列值如果不同，那么输入值肯定也不同。但是，根据同一散列函数计算出的散列值如果相同，输入值不一定相同\*\*。

什么是哈希冲突？

**当两个不同的输入值，根据同一散列函数计算出相同的散列值的现象，我们就把它叫做碰撞（哈希碰撞）** 。

HashMap的数据结构

在Java中，保存数据有两种比较简单的数据结构：数组和链表。 **数组的特点是：寻址容易，插入和删除困难；链表的特点是：寻址困难，但插入和删除容易** ；所以我们将数组和链表结合在一起，发挥两者各自的优势，使用一种叫做 **链地址法** 的方式可以解决哈希冲突：

![](https://pic2.zhimg.com/v2-6d98a3e98cf997d87de21568dc69524b_1440w.jpg)

这样我们就可以将拥有相同哈希值的对象组织成一个链表放在hash值所对应的bucket下， **但相比于hashCode返回的int类型，我们HashMap初始的容量大小**

```
DEFAULT_INITIAL_CAPACITY = 1 << 4
```

**（即2的四次方16）要远小于int类型的范围，所以我们如果只是单纯的用hashCode取余来获取对应的bucket这将会大大增加哈希碰撞的概率，并且最坏情况下还会将HashMap变成一个单链表** ，所以我们还需要对hashCode作一定的优化

hash()函数

上面提到的问题，主要是因为如果使用hashCode取余，那么相当于 **参与运算的只有hashCode的低位** ，高位是没有起到任何作用的，所以我们的思路就是让hashCode取值出的高位也参与运算，进一步降低hash碰撞的概率，使得数据分布更平均，我们把这样的操作称为 **扰动** ，在 **JDK 1.8** 中的hash()函数如下：

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);// 与自己右移16位进行异或运算（高低位异或）
}
```

这比在 **JDK 1.7** 中，更为简洁， **相比在1.7中的4次位运算，5次异或运算（9次扰动），在1.8中，只进行了1次位运算和1次异或运算（2次扰动）** ；

通过上面的 **链地址法（使用散列表）** 和 **扰动函数** 我们成功让我们的数据分布更平均，哈希碰撞减少，但是当我们的HashMap中存在大量数据时，加入我们某个bucket下对应的链表有n个元素，那么遍历时间复杂度就为O(n)，为了针对这个问题，JDK1.8在HashMap中新增了红黑树的数据结构，进一步使得遍历复杂度降低至O(logn)；

总结

简单总结一下HashMap是使用了哪些方法来有效解决哈希冲突的：

> **1\. 使用链地址法（使用散列表）来链接拥有相同hash值的数据；** **2\. 使用2次扰动函数（hash函数）来降低哈希冲突的概率，使得数据分布更平均；** **3\. 引入红黑树进一步降低遍历的时间复杂度，使得遍历更快；**

## 39.HashMap为什么不直接使用hashCode()处理后的哈希值直接作为table的下标？

答：

```
hashCode()
```

方法返回的是int整数类型，其范围为-(2 ^ 31)~(2 ^ 31 - 1)，约有40亿个映射空间，而HashMap的容量范围是在16（初始化默认值）~2 ^ 30，HashMap通常情况下是取不到最大值的，并且设备上也难以提供这么多的存储空间，从而导致通过

```
hashCode()
```

计算出的哈希值可能不在数组大小范围内，进而无法匹配存储位置；

**那怎么解决呢？**

1. HashMap自己实现了自己的hash() 方法，通过两次扰动使得它自己的哈希值高低位自行进行异或运算，降低哈希碰撞概率也使得数据分布更平均；
2. 在保证数组长度为2的幂次方的时候，使用hash() 运算之后的值与运算（&）（数组长度 - 1）来获取数组下标的方式进行存储，这样一来是比取余操作更加有效率，二来也是因为只有当数组长度为2的幂次方时，h&(length-1)才等价于h%length，三来解决了“哈希值与数组大小范围不匹配”的问题；

最近面试最多的就是被问的是Hashmap的扩展因子是0.75的问题

[从泊松分布谈起HashMap为什么默认扩容因子是0.75](https://zhuanlan.zhihu.com/p/396019103)

## 40.HashMap 的长度为什么是2的幂次方

为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀，每个链表/红黑树长度大致相同。这个实现就是把数据存到哪个链表/红黑树中的算法。

**这个算法应该如何设计呢？**

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。” 并且 采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。

**那为什么是两次扰动呢？**

答：这样就是加大哈希值低位的随机性，使得分布更均匀，从而提高对应数组存储下标位置的随机性&均匀性，最终减少Hash冲突，两次就够了，已经达到了高位低位同时参与运算的目的；

### ConcurrentHashMap篇

## 41.HashMap 和 ConcurrentHashMap 的区别

1. ConcurrentHashMap对整个桶数组进行了分割分段(Segment)，然后在每一个分段上都用lock锁进行保护，相对于HashTable的synchronized锁的粒度更精细了一些，并发性能更好，而HashMap没有锁机制，不是线程安全的。（JDK1.8之后ConcurrentHashMap启用了一种全新的方式实现,利用CAS算法。）
2. HashMap的键值对允许有null，但是ConCurrentHashMap都不允许。

## 42.ConcurrentHashMap 和 Hashtable 的区别？

ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。

- **底层数据结构** ： JDK1.7的 ConcurrentHashMap 底层采用 **分段的数组+链表** 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 **数组+链表** 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；
- **实现线程安全的方式（重要）** ： ① **在JDK1.7的时候，ConcurrentHashMap（分段锁）** 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。（默认分配16个Segment，比Hashtable效率提高16倍。） **到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6以后 对 synchronized锁做了很多优化）** 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；② **Hashtable(同一把锁)**:使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

**两者的对比图** ：

HashTable:

![](https://pic1.zhimg.com/v2-30219d2251b3ceb5fca8810b8529190e_1440w.jpg)

JDK1.7的ConcurrentHashMap：

![](https://picx.zhimg.com/v2-c0b6462e5d2dee5c54ccf7b4fb9fe657_1440w.jpg)

JDK1.8的ConcurrentHashMap（TreeBin: 红黑二叉树节点 Node: 链表节点）：

![](https://pic2.zhimg.com/v2-a9b14f02bc6585114f3a8a645eaedf9d_1440w.jpg)

答：ConcurrentHashMap 结合了 HashMap 和 HashTable 二者的优势。HashMap 没有考虑同步，HashTable 考虑了同步的问题。但是 HashTable 在每次同步执行时都要锁住整个结构。 ConcurrentHashMap 锁的方式是稍微细粒度的。

## 43\. ConcurrentHashMap 底层具体实现知道吗？实现原理是什么？

**JDK1.7**

首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

在JDK1.7中，ConcurrentHashMap采用Segment + HashEntry的方式进行实现，结构如下：

一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和HashMap类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

![](https://pic2.zhimg.com/v2-01a90acce9535be2965ff94f6bf6976f_1440w.jpg)

1. 该类包含两个静态内部类 HashEntry 和 Segment ；前者用来封装映射表的键值对，后者用来充当锁的角色；
2. Segment 是一种可重入的锁 ReentrantLock，每个 Segment 守护一个HashEntry 数组里得元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment 锁。

**JDK1.8**

在 **JDK1.8中，放弃了Segment臃肿的设计，取而代之的是采用Node + CAS + Synchronized来保证并发安全进行实现** ，synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发，效率又提升N倍。

结构如下：

![](https://picx.zhimg.com/v2-5da38d9bcba147adb3bf3b7b6a20db7d_1440w.jpg)

**附加源码，有需要的可以看看**

插入元素过程（建议去看看源码）：

如果相应位置的Node还没有初始化，则调用CAS插入相应的数据；

```
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
}
```

如果相应位置的Node不为空，且当前该节点不处于移动状态，则对该节点加synchronized锁，如果该节点的hash不小于0，则遍历链表更新节点或插入新节点；

```
if (fh >= 0) {
    binCount = 1;
    for (Node<K,V> e = f;; ++binCount) {
        K ek;
        if (e.hash == hash &&
            ((ek = e.key) == key ||
             (ek != null && key.equals(ek)))) {
            oldVal = e.val;
            if (!onlyIfAbsent)
                e.val = value;
            break;
        }
        Node<K,V> pred = e;
        if ((e = e.next) == null) {
            pred.next = new Node<K,V>(hash, key, value, null);
            break;
        }
    }
}
```
1. 如果该节点是TreeBin类型的节点，说明是红黑树结构，则通过putTreeVal方法往红黑树中插入节点；如果binCount不为0，说明put操作对数据产生了影响，如果当前链表的个数达到8个，则通过treeifyBin方法转化为红黑树，如果oldVal不为空，说明是一次更新操作，没有对元素个数产生影响，则直接返回旧值；
2. 如果插入的是一个新节点，则执行addCount()方法尝试更新元素个数baseCount；

## 44.Java中的另一个线程安全的与HashMap极其类似的类是什么?同样是线程安全,它与HashTable在线程同步上有什么不同?

```
ConcurrentHashMap
```

类(是

```
Java
```

并发包

```
java.util.concurrent
```

中提供的一个线程安全且高效的

```
HashMap
```

实现)。

```
HashTable
```

是使用

```
synchronize
```

关键字加锁的原理(就是对对象加锁);

而针对

```
ConcurrentHashMap
```

,在

```
JDK1.7
```

中采用分段锁的方式;

```
JDK1.8
```

中直接采用了

```
CAS
```

**(无锁算法)+**

```
synchronized
```

## 45.HashMap&ConcurrentHashMap的区别?

除了加锁,原理上无太大区别。另外,

```
HashMap
```

的键值对允许有

```
null
```

,但是

```
ConCurrentHashMap
```

都不允许。

补充： 转红黑树的阈值为什么是8？红黑树转链表为什么是6？

## 46.为什么ConcurrentHashMap比HashTable效率要高?

```
HashTable
```

使用一把锁(锁住整个链表结构)处理并发问题,多个线程竞争一把锁,容易阻塞;

```
ConcurrentHashMap
```
- JDK1.7 中使用分段锁(ReentrantLock +Segment +HashEntry ),相当于把一个HashMap 分成多个段,每段分配一把锁,这样支持多线程访问。锁粒度:基于Segment,包含多个HashEntry 。
- JDK1.8 中使用CAS +synchronized +Node +红黑树。锁粒度:Node (首结点)(实现Map.Entry<K,V> )。锁粒度降低了。

## 47.针对ConcurrentHashMap锁机制具体分析(JDK1.7VSJDK1.8)?

```
JDK1.7
```

中,采用分段锁的机制,实现并发的更新操作,底层采用数组+链表的存储结构,包括两个核心静态内部类

```
Segment
```

和

```
HashEntry
```

。

- ①、 Segment 继承ReentrantLock (重入锁)用来充当锁的角色,每个Segment 对象守护每个散列映射表的若干个桶;
- ②、 HashEntry 用来封装映射表的键-值对;
- ③、每个桶是由若干个 HashEntry 对象链接起来的链表;
![](https://picx.zhimg.com/v2-2370d580c2b6534ed36c2a94251d3649_1440w.jpg)

```
JDK1.8
```

中,采用

```
Node
```

+

```
CAS
```

+

```
Synchronized
```

来保证并发安全。取消类

```
Segment
```

,直接用

```
table
```

数组存储键值对;当

```
HashEntry
```

对象组成的链表长度超过

```
TREEIFY_THRESHOLD
```

时,链表转换为红黑树,提升性能。底层变更为数组+链表+红黑树。

![](https://pica.zhimg.com/v2-edf8fef1ba7140d2eaaf9ecf0e2b15e0_1440w.jpg)

## 48.ConcurrentHashMap在JDK1.8中,为什么要使用内置锁synchronized来代替重入锁ReentrantLock?

- ①、粒度降低了;
- ②、 JVM 开发团队没有放弃synchronized,而且基于JVM 的synchronized 优化空间更大,更加自然。
- ③、在大量的数据操作下,对于 JVM 的内存压力,基于API 的ReentrantLock 会开销更多的内存。

## 49.ConcurrentHashMap简单介绍?

- ①、重要的常量:
- private transient volatile intsizeCtl;
- 当为负数时,-1表示正在初始化,-N表示N-1个线程正在进行扩容;
- 当为0时,表示 table 还没有初始化;
- 当为其他正数时,表示初始化或者下一次进行扩容的大小。
- ②、数据结构:
- Node 是存储结构的基本单元,继承HashMap 中的Entry,用于存储数据;
- TreeNode 继承Node,但是数据结构换成了二叉树结构,是红黑树的存储结构,用于红黑树中存储数据;
- TreeBin 是封装TreeNode 的容器,提供转换红黑树的一些条件和锁的控制。
- ③、存储对象时( put() 方法):
- 1.如果没有初始化,就调用 initTable() 方法来进行初始化;
- 2.如果没有 hash 冲突就直接CAS 无锁插入;
- 3.如果需要扩容,就先进行扩容;
- 4.如果存在 hash 冲突,就加锁来保证线程安全,两种情况:一种是链表形式就直接遍历到尾端插入,一种是红黑树就按照红黑树结构插入;
- 5.如果该链表的数量大于阀值8,就要先转换成红黑树的结构, break 再一次进入循环
- 6.如果添加成功就调用 addCount() 方法统计size,并且检查是否需要扩容。
- ④、 **扩容方法** **transfer()** **:** 默认容量为16,扩容时,容量变为原来的两倍。 **helpTransfer()** **:** 调用多个工作线程一起帮助进行扩容,这样的效率就会更高。
- ⑤、获取对象时( get() 方法):
- 1.计算hash值,定位到该 table 索引位置,如果是首结点符合就返回;
- 2.如果遇到扩容时,会调用标记正在扩容结点 ForwardingNode.find() 方法,查找该结点,匹配就返回;
- 3.以上都不符合的话,就往下遍历结点,匹配就返回,否则最后就返回 null 。

## 50\. ConcurrentHashMap的并发度是什么?

程序运行时能够同时更新

```
ConccurentHashMap
```

且不产生锁竞争的最大线程数。默认为16,且可以在构造函数中设置。当用户设置并发度时,

```
ConcurrentHashMap
```

会使用大于等于该值的最小2幂指数作为实际并发度(假如用户设置并发度为17,实际并发度则为32)

当然我之前也给大家整理了一些Java面试题高级和初中级的都有：

中高级面试题： [zhuanlan.zhihu.com/p/34](https://zhuanlan.zhihu.com/p/346994550)

基础一点的面试题：

[zhuanlan.zhihu.com/p/37](https://zhuanlan.zhihu.com/p/379726770)

我是架构师小于哥关注

[@终端研发部](https://www.zhihu.com/people/0562ae57ba411b146099313075609e94)

我会偶尔写写代码，出来聊聊天，专注职场经验和开发小技巧的分享

参考：

编辑于 2022-04-29 09:16