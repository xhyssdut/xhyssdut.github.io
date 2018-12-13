---
layout: post
title:  "Java集合（三）：ArrayDeque，ArrayList"
date:   2018-11-21
desc: "Java集合（三）：ArrayDeque，ArrayList,ArrayList实现了List的所有方法，并"
keywords: "Java,Collection,ArrayDeque,ArrayList"
categories: [Java]
tags: [Java,Collection]
icon: icon-html
---
## ArrayDeque
`ArrayDeque` 底层是使用数组存储元素，同时还使用了两个索引来表征当前数组的状态，分别是 `head` 和 `tail`。`head` 是头部元素的索引，但注意 `tail` 不是尾部元素的索引，而是尾部元素的下一位，即下一个将要被加入的元素的索引。
### 初始化
`ArrayDeque` 提供了三个构造方法，分别是默认容量，指定容量及依据给定的集合中的元素进行创建。默认容量为16。
`ArrayDeque` 对数组的大小(即队列的容量)有特殊的要求，必须是 2^n。通过 `allocateElements` 方法计算初始容量：
``` java
private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    //通过五步右移，使容量成为2的n次方
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;
        if (initialCapacity < 0)   // 初始大小大于int最大值
            initialCapacity >>>= 1;// 右移一位
    }
    elements = new Object[initialCapacity];
}
```
### 添加元素
向末尾添加元素：
``` java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    //使用环形索引，因为elements.length - 1全为1，实现了环形索引的计算
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
    }
```
向头部添加元素：
``` java
public void addFirst(E e) {
    if (e == null) //不支持值为null的元素
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}
```
### 扩容
``` java
private void doubleCapacity() {
    assert head == tail; //扩容时头部索引和尾部索引肯定相等
    int p = head;
    int n = elements.length;
    //头部索引到数组末端(length-1处)共有多少元素
    int r = n - p; // number of elements to the right of p
    //容量翻倍
    int newCapacity = n << 1;
    //容量过大，溢出了
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    //分配新空间
    Object[] a = new Object[newCapacity];
    //复制头部索引到数组末端的元素到新数组的头部
    System.arraycopy(elements, p, a, 0, r);
    //复制其余元素
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    //重置头尾索引
    head = 0;
    tail = n;
}
```
### 移除元素
ArrayDeque支持从头尾两端移除元素，remove方法是通过poll来实现的。因为是基于数组的，在了解了环的原理后这段代码就比较容易理解了。
``` java
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result == null)
        return null;
    elements[h] = null;     // Must null out slot
    head = (h + 1) & (elements.length - 1);
    return result;
}
public E pollLast() {
    int t = (tail - 1) & (elements.length - 1);
    @SuppressWarnings("unchecked")
    E result = (E) elements[t];
    if (result == null)
        return null;
    elements[t] = null;
    tail = t;
    return result;
}
private boolean delete(int i) {
    checkInvariants();
    final Object[] elements = this.elements;
    final int mask = elements.length - 1;
    final int h = head;
    final int t = tail;
    final int front = (i - h) & mask;
    final int back  = (t - i) & mask;
    // Invariant: head <= i < tail mod circularity
    if (front >= ((t - h) & mask))
        throw new ConcurrentModificationException();
    // Optimize for least element motion
    if (front < back) {
        if (h <= i) {
            System.arraycopy(elements, h, elements, h + 1, front);
        } else { // Wrap around
            System.arraycopy(elements, 0, elements, 1, i);
            elements[0] = elements[mask];
            System.arraycopy(elements, h, elements, h + 1, mask - h);
        }
        elements[h] = null;
        head = (h + 1) & mask;
        return false;
    } else {
        if (i < t) { // Copy the null tail as well
            System.arraycopy(elements, i + 1, elements, i, back);
            tail = t - 1;
        } else { // Wrap around
            System.arraycopy(elements, i + 1, elements, i, mask - i);
            elements[mask] = elements[0];
            System.arraycopy(elements, 1, elements, 0, t);
            tail = (t - 1) & mask;
        }
        return true;
    }
}
```

## ArrayList
ArrayList实现了List的所有方法，并且提供了一个保存所有元素（包括空）的集合，它和Vector是基本一致的，处理ArrayList是非同步类，由于ArrayList的方法都比较简单，于是在这里不进行太多说明。
下面是一些感兴趣的方法：
``` java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r, elementData, w, size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

对于ArrayList，它的特点是：内部采用动态数组实现，这决定了：
- 可以随机访问，按照索引位置进行访问效率很高，用算法描述中的术语，效率是O(1)，简单说就是可以一步到位。
- 除非数组已排序，否则按照内容查找元素效率比较低，具体是O(N)，N为数组内容长度，也就是说，性能与数组长度成正比。
- 添加元素的效率还可以，重新分配和拷贝数组的开销被平摊了，具体来说，添加N个元素的效率为O(N)。
- 插入和删除元素的效率比较低，因为需要移动元素，具体为O(N)。 
ArrayList与Vector的区别
1）  Vector的方法都是同步的(Synchronized),是线程安全的(thread-safe)，而ArrayList的方法不是，由于线程的同步必然要影响性能，因此,ArrayList的性能比Vector好。 
2） 当Vector或ArrayList中的元素超过它的初始大小时,Vector会将它的容量翻倍,而ArrayList只增加50%的大小，这样,ArrayList就有利于节约内存空间。
