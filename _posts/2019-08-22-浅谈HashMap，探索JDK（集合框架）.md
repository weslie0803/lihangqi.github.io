---
layout:     post   				    # 使用的布局（不需要改）
title:      浅谈HashMap，探索JDK（集合框架）				# 标题 
subtitle:   Collection API 位于 java.util 包中。包中的 Collection 接口是 JAVA 对于集合这一概念的抽象，存储一组类型相同的对象。  #副标题
date:       2019-08-22				# 时间
author:     凌洛 						# 作者
header-img: img/post-bg-macbook1.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
---

Collection API 位于 java.util 包中。包中的 Collection 接口是 JAVA 对于集合这一概念的抽象，存储一组类型相同的对象。
还有一个很重要的接口：Iterable，Collection 接口以继承的方式对 Iterable 做了扩展。实现 Collection 接口的类可以获得增强 for 循环（forEach）。

# 数据结构（数组+链表)

HashMap 是 JAVA 集合框架的成员。基于 [ 数组 + 链表 ] 的数据结构存储 key-value 形式的数据。key 是每条数据的唯一标识，HashMap 通过一个 hash 算法（也称散列算法）根据 key 值计算出这条数据在数组中的位置，即数组下标，然后把数据装载到一个链表元素```Node<K, V>```中，最后根据数组下标进行落桶（bucket）操作。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/e0ec9574fa0fa8bb78d6062152f7b3b9eecf2355.png)
## hash碰撞（冲突）：

如果两个输入的 hash 结果相同，则称这两个输入是一个碰撞(Collision)。

在JAVA中，采用“链地址法”解决 hash碰撞。HashMap 在数组中存放第一个落桶的节点，这个节点也是链表的 head节点，拥有一个 next 属性指向 null，当下一个相同 hash 值的元素落桶，则使此 head节点的 next 指向新的元素，即后来的节点作为链表的 tail节点。

由上图可知，数组在 0 和 2 的位置存放了节点k1/k2，当节点 k3 与 k2 发生了 hash碰撞，则使节点 k2 的 next 指向节点 k3。值得注意的是，在 JAVA8 中，为了提高检索效率，当链表的节点数量超过8个，并且整个数组容量超过 64 个，则把这个链表重载成红黑树（树化），否则进行 2 倍扩容并且重新散列（rehash）所有节点。树化操作是因为链表的检索是线性时间O(n)，而红黑树是对数时间O(lgn)。这么处理大概是为了尽可能避免过早的把数据存放到桶外（形成长链表），因为桶数组的容量是参与元素索引计算的。

# 存储(hash算法，hash冲突，初始化，扩容)

```java
// 对用户提供的put方法
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
// 散列（hash）方法
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 这里的位异或运算和无符号右移运算后边会详细说明[1]
}
// 实现Map.put
// 如果对象已存在返回上一个值，如果没有则返回null
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // 这里的table就是HashMap的桶数组，这个数组是需要制定容量的，默认16，属性"DEFAULT_INITIAL_CAPACITY"
        n = (tab = resize()).length; // HashMap在第一次put的时候进行初始化
    if ((p = tab[i = (n - 1) & hash]) == null) // 判断即将落桶的位置是否已经有Node存在，即是否存在hash冲突，这里的位与运算后边会详细说明[2]
        tab[i] = newNode(hash, key, value, null); // 不存在hash冲突直接落桶
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // 判断是否在数组中存放的对象（即链表头节点）与新的对象的key值相同，如果相同直接提取到新的拷贝e中供后续操作
            e = p;
        else if (p instanceof TreeNode) // 如果p处于红黑树中，则调用TreeNode.putTreeVal()方法提取旧节点到e中供后续操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else { // 如果p处于链表中
            for (int binCount = 0; ; ++binCount) { // 遍历链表
                if ((e = p.next) == null) { // 当整个链表不存在与新节点相同的key，则直接把新节点加入到链表的尾部
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // 当链表元素数量到达指定阈值，默认8个，进行“树化”
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break; // 当找到与新节点相同的key，提取到e中供后续操作
                p = e;
            }
        }
        if (e != null) { // 当新节点的key已经存在（这里就是上边多处提到的“供后续操作”）
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) // onlyIfAbsent参数的意思是“是否不覆盖旧值”
                e.value = value;
            afterNodeAccess(e); // 这个方法是为了继承HashMap的LinkedHashMap类服务的，可以忽略
            return oldValue;
        }
    }
    ++modCount; // 这是一个记录操作次数的变量，后边会详细说明[3]
    if (++size > threshold) // 如果不是值覆盖会执行到这步，如果本次元素插入导致了桶数量超过阈值，则进行扩容，后边会详细说明[4]
        resize();
    afterNodeInsertion(evict); // 和afterNodeAccess()方法一样，可以忽略
    return null;
}
// 初始化或加倍表格大小
// 如果为null，则分配初始容量。否则，进行2次幂扩展。
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // 旧的数组容量
    int oldThr = threshold; // 旧的扩容阈值
    int newCap, newThr = 0; // 声明新的数组容量和扩容阈值
    if (oldCap > 0) { // 旧的数组容量大于0说明本次是扩容操作
        if (oldCap >= MAXIMUM_CAPACITY) { // 扩容前的数组大小如果已经达到最大(2^30)了
            threshold = Integer.MAX_VALUE; // 修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && // 普通扩容，在下边展开解释[4]
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 新的扩容阈值同样扩大2倍
    }
    else if (oldThr > 0) // 这里话多了，在下边解释[5]
        newCap = oldThr;
    else { // zero initial threshold signifies using defaults, 这里和 [5] 一起看吧
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) { // 计算新的扩容阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 删除了旧节点的引用（很细节）
                if (e.next == null) // 当桶里只有一个节点，重新计算索引位置落桶
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 创建两条链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) { // 直接用容量与 hash 位与运算，相当于舍掉低位特征，大概是针对hash冲突进行一次随机散列。结果为0的保持原来的索引位置不变
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else { // 否则向右偏移旧数组容量
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
                        newTab[j + oldCap] = hiHead; // 向右偏移旧数组容量
                    }
                }
            }
        }
    }
    return newTab;
}
```
# 详细说明

*[1]*  
```
(h = key.hashCode()) ^ (h >>> 16)
public native int hashCode(); 
```
这里的 hashCode() 是一个 native 方法，根据一定的规则将与对象相关的信息（比如对象的存储地址，对象的字段等）映射成一个数值，这个数值称作为散列值。

无符号右移：>>>

按二进制形式把所有的数字向右移动对应的位数，低位移出(舍弃)，（如果是“有符号右移>>”：高位的空位补符号位，即正数补0，负数补1。当运算数是 byte 或 short 类型时，将自动把这些类型扩大为 int 型。由于 int 类型是 32 位，这里的右移16位（舍去低位）并与 hashCode 异或运算将导致高位的影响传播到低位。

*[2]*  
```
i = (n - 1) & hash
n = table.length
```
此处是根据hash值计算数组下标。n 是容量，值得注意的是，HashMap的默认初始容量是16，指定容量也会被扩展到2的幂次，归根结底就是为了在计算数组下标的时候，用 (n - 1) 与 hash值进行位与运算。因为在[1]中计算hash值的计算中，把高位的特征传播到了低位，而2的幂次减一的值永远是形如：1111，11111，111111的，因此index仅与hash值的低n位有关，hash值的高位都被与操作置为了0。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/8b855a731aa40379ced49e9ab781486375745611.png)
上图为 HashMap 根据 key 计算 hash 值并最终计算出数组下标 index 的过程。我们来验证一下：

**1.创建一个 HashMap：**

 可以看到，在创建map这一步，内部已经做了初始化如：size、modCount、threshold、loadFactor

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/9d4f624dc3cd74792aba88ba561049cc29db4d5e.png)
**2.执行下一步，put 一个 key 为 "WANGNIMA" 的元素：**

现在数组 table 中已经有了数据，包括 size、modCount、threshold 也都有了值。
 这里解释一下 threshold 这个变量，它作为 HashMap 扩容的阈值，在初始化的时候，是根据 ```loadFactor（加载因子，默认0.75f）* initialCapacity（初始容量，默认16）```得到的，即``` 0.75  *  16 = 12```，当数组 size 超过这个阈值的时候，触发 2 倍的扩容。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/a439f51929303902b1ef10d372afa0049299b921.png)
**3.接下来继续执行到把两个元素都 put 进去：**

看到```WANGNIMA```被如期放到了下标 0 的位置，```WANGNIMA2```被放到了 9 的位置。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/92e38bca3d785591a46f93cc1ba7beb8cb3708cc.png)
**4.测试 hash 冲突：**

我们找到了与 ```WANGNIMA```的 hashCode 值相同的字符串 ```Z@]LRTyvHV\\SCV^```，由图可知：table 中还是 0 和 9 的位置有元素，最后 put 的```Z@]LRTyvHV\\SCV^``` 被放在了 ```WANGNIMA``` 的 next 中，也就是链表的第二个位置处。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/fa1b55884eeddd4507d4bfbbd9d5e291bf565c60.png)

*[3]*  ```++modCount;```
 fail-fast 机制。
开篇说道：实现 Collection 接口的类可以获得增强 for 循环。其实在编译阶段，编译器会把 forEach 这样的语句编译成迭代器迭代的方式：

```java
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>(16);
        map.put("WANGNIMA", 250);
        map.put("WANGNIMA2", 250);
        for (String key : map.keySet()) {
            System.out.println("Key = " + key);
        }
    }
    // .class
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap(16);
        map.put("WANGNIMA", 250);
        map.put("WANGNIMA2", 250);
        Iterator var2 = map.keySet().iterator();

        while(var2.hasNext()) {
            String key = (String)var2.next();
            System.out.println("Key = " + key);
        }

    }
```
如果在创建迭代器之后的任何时候对 map 进行结构修改，除了迭代器自己以外的任何线程调用 remove() 方法，迭代器将抛出 ConcurrentModificationException，因此，在并发修改的情况下，迭代器会快速失败，而不是冒着非确定性行为的风险。  

*[4]* 

```
else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
    newThr = oldThr << 1;
```
其中```oldCap << 1``` 相当于 oldCap*2，即把新的数组容量 "newCap" 扩大2倍。

*[5]*

```java
else if (oldThr > 0) // 旧的扩容阈值赋值给新的数组容量

    newCap = oldThr;

else { // zero initial threshold signifies using defaults

    newCap = DEFAULT_INITIAL_CAPACITY;

    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);

}
```

首先 HashMap 有三个构造方法，HashMap()  / HashMap(int initialCapacity) / HashMap(int initialCapacity, float loadFactor)
无参构造仅仅初始化了 loadFactor 为默认的 0.75f；
HashMap(int initialCapacity) 构造也是调用了 HashMap(int initialCapacity, float loadFactor)，而把DEFAULT_LOAD_FACTOR 缺省传入了，这两个构造方法初始化了 loadFactor 和 threshold，值得注意的是，这里的 threshold 代表的却是数组容量，将会在首次 put 操作的时候，作为数组初始化的容量值，然后再去乘 loadFactor 作为真正意义上的 "扩容阈值"。

可见，为数组申请内存空间这个工作被分配给了首次 put 操作而非构造方法，当 oldThr > 0 就说明用户调用了有参构造方法（指定了初始容量，并被构造方法 "缓存" 到了threshold中了），需要初始化一个 threshold 大小的数组，即 newCap；否则，初始化的数组容量为缺省的 16，初始化的扩容阈值为缺省的 16 * 0.75。

# 总结
 数组和链表是 JAVA 中最常见的两种数据结构，HashMap 的设计者巧妙的利用了这两个数据结构的优点。在散列算法的实现上，权衡了空间、时间和算法复杂度。
 要注意的是：HashMap 的扩容操作是十分耗费时间和空间的，所以建议开发者在应用中，一定要设置初始容量，防止动态扩容的发生。这一点在《阿里巴巴Java开发手册》中也有提到:

![阿里巴巴Java开发手册](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/1932facedc3be10ed2caa9df3695fbdfa30c4c1d.jpeg)