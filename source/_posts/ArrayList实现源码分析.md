title: ArrayList实现源码分析
date: 2018-01-20 22:24:32
tags: [Java,源码解读]
categories: Java
---

## ArrayList简介
ArrayList就是披着list外衣的动态数组，相比于Linkedlist随机访问更快。值得注意的是ArrayList不是线程安全的。
## 内部字段
```
// 初始化时的默认容量大小
private static final int DEFAULT_CAPACITY = 10;
// 空列表的数组, 即capacity为0时的内部数组
private static final Object[] EMPTY_ELEMENTDATA = {};
// 当使用List list = new ArrayList()即使用默认容量大小时创建的列表内部的数组，目的是为了节省空间，当第一次add时会进行膨胀替换。
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
transient Object[] elementData; // non-private to simplify nested class access
// 当前列表的元素的数量
private int size;
```

## 构造函数
```
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## 增

```
    public boolean add(E e) {
        //确保数组能容下e
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //赋给数组最后位置 数组大小+1
        elementData[size++] = e;
        return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
    	// 空数组 最小容量
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) 	{
        modCount++;

        // overflow-conscious code
        // 要求的最小容量比现在数组大小大 必须得扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 扩容后的大小为原来容量大小1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 指定的最小容量 > 1.5 * oldCapacity 扩容后的大小为指定的最小容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 数组拷贝
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) 	{
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
增加数组的核心在于扩容，目前的扩容策略是分配1.5倍原来数组大小的新数组。

## 删
```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```
删除就是index位置之后的元素往前挪，数组最后位置置空。删除并不会主动缩容。

## 查
```
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

## 改
```
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

## 参考

+ JDK8源码