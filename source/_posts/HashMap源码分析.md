title: HashMap源码分析
date: 2018-01-20 22:21:10
tags: [Java,源码解读]
categories: Java
---
## HashMap简介
HashMap是JDK中Map的最常用的实现类。HashMap使用hash算法进行数据的存储和查询。内部使用一个Entry表示键值对key-value。用Entry的数组保存所有键值对, Entry通过链表的方式链接后续的节点(1.8后会根据链表长度决定是否转换成一棵树类似TreeMap来节省查询时间) Entry通过计算key的hash值来决定映射到具体的哪个数组（也叫Bucket)中。
另外HashMap种还涉及到key-value的存储查询操作以及扩容操作。存储和查询依赖key的hashCode()和equals()方法，这也是EffectiveJava中推荐为每个类都override这两个方法的原因。

## 源码解读
HashMap中有三个关键的概念值capacity、threshold和loadFactor。capacity表示内部数组的数组大小。
threshold作为HashMap扩容的依据，当map中的key-value键值对的数量大于threshold时需要进行扩容。
当不显示声明时，默认初始capacity为16, loadFactory为0.75。
扩容时(resize)，会创建一个当前的capacity的两倍大小的数组，然后将旧的数据放到新数组中，最后替换新旧数组完成扩容。
HashMap具有多个构造器。

```
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
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

### 扩容机制
#### put
扩容主要由put方法引发，put方法的作用是增加一对key-value，如果已经存在相同的key（使用equals判断逻辑相等)则替换为新的value。
put主要调用了putVal
putVal的处理过程为

1. 如果table没有使用过的情况，则进行一次resize
2. 计算key的hash值，然后获取底层table数组的第(n-1)&hash的位置的数组数据，即hash对n取模的位置，依赖的是n为2的次方这一条件
3. 先检查该bucket第一个元素是否是和插入的key相等(如果是同一个对象则肯定equals)
4. 如果不相等并且是TreeNode的情况，调用TreeNode的put方法
否则循环遍历链表，如果找到相等的key跳出循环否则达到最后一个节点时将5. 新的节点添加到链表最后, 当前面找到了相同的key的情况下替换这个节点的value为新的value。
6. 最后如果新增了key-value对，则增加size并且判断是否超过了threshold,如果超过则需要进行resize扩容

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果table没有使用过的情况，则进行一次resize
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 获取底层table数组的第(n-1)&hash的位置的数据，即hash对n取模，依赖的是n为2的次方这一条件
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果这个bucket的元素还是null，则创建新节点并设置到table数组中
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 先检查第一个元素是否是和插入的key相等(如果是同一个对象则肯定equals)
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是TreeNode的情况，则使用TreeNode的put方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 其他情况下遍历后续链表节点
            for (int binCount = 0; ; ++binCount) {
                // 如果遍历到了最后一个节点，说明没有匹配的key，则创建一个新的节点并添加到最后
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果超过了转换成树结构的阈值，则将链表转换成树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果找到了和key相等的节点跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 已经存在该key的情况时，将对应的节点的value设置为新的value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
#### resize
resize负责初始化或者扩容内部数组，通过将数组大小扩大一倍。
resize大体过程为

首先计算resize()后的新的capacity和threshold值。如果原有的capacity大于零则将capacity增加一倍，否则设置成默认的capacity。
创建新的数组，大小是新的capacity
将旧数组的元素放置到新数组中

```
final Node<K,V>[] resize() {
    // 将字段引用copy到局部变量表，这样在之后的使用时可以减少getField指令的调用。
    Node<K,V>[] oldTab = table;
    // oldCap为原数组的大小或当空时为0
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新的数组的大小是旧数组的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
                 // 当旧的的数组大小大于等于默认大小时，threshold也扩大一倍。
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
    // 按照新的capacity创建新数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 将原数组中的数组复制到新数组中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                // 如果e是该bucket唯一的一个元素，则直接赋值到新数组中。
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                // TreeNode的情况则使用TreeNode中的split方法将这个树分成两个小树
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 否则则创建两个链表用来存放要放的数据，hash值&oldCap为0的(即oldCap的1的位置的和hash值的同样的位置都是1，同样是基于capacity是2的次方这一前提)为low链表，反之为high链表, 通过这种方式将旧的数据分到两个链表中再放到各自对应余数的位置。
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 按照e.hash值区分放在loTail后还是hiTail后
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
                    // 处理完之后放到新数组中
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
### HashMap红黑树实现
基本结构

```
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        ...
}

```

## HashSet
讲完HashMap就可以简单提一下HashSet了，HashSet内部使用一个HashMap来保存数据，因为只需要保存key,所有的value都是相同的一个值。

## 参考
+ JDK8 源码