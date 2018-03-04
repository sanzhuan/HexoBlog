title: LinkedList源码分析
date: 2018-01-20 22:24:20
tags: [Java,源码解读]
categories: Java
---
## LinkedList简介
LinkedList是双链表实现。值得注意的是LinkedList不是线程安全的。

## 构造函数与内部类


```
transient int size = 0;
/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;
/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;
	
	private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
	
    public LinkedList() {}
    
	public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

## 增

```
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
    void linkLast(E e) {
        final Node<E> l = last;
        // new 一个新节点 pre为l next为 null
        final Node<E> newNode = new Node<>(l, e, null);
        // 尾节点指向新创建的节点
        last = newNode;
        // 如果之前链表为空 则头节点也指向新创建的节点
        if (l == null)
            first = newNode;
        else
        	// 否则把新节点挂在前节点后面
            l.next = newNode;
        size++;
        modCount++;
    }
```

## 删
```
    public E remove() {
        return removeFirst();
    }
    
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
    
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        //头节点指向一个节点
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
## 查
```
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    
    Node<E> node(int index) {
        // assert isElementIndex(index);
		// index < size / 2 从头节点开始往后遍历 否则从尾节点往前遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
查找做了小优化，如果index < size / 2，则从头节点开始往后遍历，否则从尾节点往前遍历。

## 改
```
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```

## 参考
+ JDK8 源码
    
    
    