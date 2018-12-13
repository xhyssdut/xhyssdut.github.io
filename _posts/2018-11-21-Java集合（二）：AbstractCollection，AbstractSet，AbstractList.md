---
layout: post
title:  "Java集合（二）：AbstractCollection，AbstractSet，AbstractList"
date:   2018-11-21
desc: "Java集合（二）：AbstractCollection，AbstractSet，AbstractList,下面介绍一下AbstractCollection ———— Collection接口的实现抽象类，也是大部分类的最上级父类，下文主要介绍一下它的具体实现。
"
keywords: "Java,Collection,Map,Set,Queue"
categories: [Java]
tags: [Java,Collection]
icon: icon-html
---
下面介绍一下AbstractCollection ———— Collection接口的实现抽象类，也是大部分类的最上级父类，下文主要介绍一下它的具体实现。

* * *
由于迭代器以及size()方法与集合的数据存储方式有关，所有`AbstractCollection`没有实现迭代器和size()方法。

下面罗列一下有具体实现并且比较浅显的方法：
``` java
/**contains方法对null和非null值做了分别处理**/
public boolean contains(Object o)
/**通过contains方法进行处理**/
public boolean containsAll(Collection<?> c)
/**通过add方法进行处理**/
public boolean addAll(Collection<? extends E> c)
/**调用了迭代器的remove方法**/
public boolean remove(Object o)
public boolean removeAll(Collection<?> c)
public boolean retainAll(Collection<?> c)
public void clear()
```

其中`toArray()`方法比较复杂，在下面进行讲解
``` java
public Object[] toArray() {
    // 创建新空间，在创建过程中可能会有大小的变化    
    Object[] r = new Object[size()];
    Iterator<E> it = iterator();
    for (int i = 0; i < r.length; i++) 
        //当实际大小小于估计的大小时，将这个数组进行裁剪然后返回。
        if (! it.hasNext())
            return Arrays.copyOf(r, i);
        r[i] = it.next();
    }
    //当实际大小大于估计大小时，进行下一个方法对数组进行扩充。
    return it.hasNext() ? finishToArray(r, it) : r;
}  

private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
    //设置初始值，因为r初始是满的。
    int i = r.length;
    while (it.hasNext()) {
        int cap = r.length;
        if (i == cap) {
            //对数组进行扩充到自己的1.5倍+1
            int newCap = cap + (cap >> 1) + 1;
            // 如果大于最大值了，进行大数字校验，可能会报错OOM
            if (newCap - MAX_ARRAY_SIZE > 0)
                newCap = hugeCapacity(cap + 1);
            //进行数组扩充
            r = Arrays.copyOf(r, newCap);
        }
        r[i++] = (T)it.next();
    }
    // 如果扩充之后的数组过大了，就得进行裁剪
    return (i == r.length) ? r : Arrays.copyOf(r, i);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError("Required array size too large");
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :MAX_ARRAY_SIZE;
}

```
## AbstractSet
`AbstractSet`是`AbstractCollection`的子类，它并没有继承任何`AbstractCollection`的方法，只重写了`equals`和`hashCode`方法，其中`equals`方法使用的`containsAll`方法。
## AbstractList
`AbstractList`提供了一套**随机读取**的基本实现，对于链表之类的线性读取，可以继承`AbstractSequentialList`来进行二次开发。这个类实现了单向迭代器`iterator`和双向迭代器`listIterator`
下面会介绍一些需要精读的方法。
``` java
    private class Itr implements Iterator<E> {
        //迭代器当前所在的位置,最小时为0，最大值为size()
        int cursor = 0;
        //上一次调用next或者previous方法时迭代器所在的位置,初始化为-1
        int lastRet = -1;
        //迭代器创建时的修改次数，和线程安全有关系
        int expectedModCount = modCount;
        public boolean hasNext() {
            return cursor != size();
        }
        //获取迭代器所在的元素，并将指针后移
        public E next() {
            //判断集合是否被修改
            checkForComodification();
            try {
                int i = cursor;
                E next = get(i);
                lastRet = i;
                cursor = i + 1;
                return next;
            } catch (IndexOutOfBoundsException e) {
                checkForComodification();
                throw new NoSuchElementException();
            }
        }
        //移除当前元素
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
            try {
                AbstractList.this.remove(lastRet);
                //这里的操作，保证了向前走的指针也不会出现逻辑错误
                if (lastRet < cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException e) {
                throw new ConcurrentModificationException();
            }
        }
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            cursor = index;
        }
        public boolean hasPrevious() {
            return cursor != 0;
        }
        //获取迭代器之前所在的元素，并将指针前移
        public E previous() {
            checkForComodification();
            try {
                int i = cursor - 1;
                E previous = get(i);
                lastRet = cursor = i;
                return previous;
            } catch (IndexOutOfBoundsException e) {
                checkForComodification();
                throw new NoSuchElementException();
            }
        }
        //在迭代器的当时位置进行修改元素
        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
            try {
                AbstractList.this.set(lastRet, e);
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
        //在当前位置添加一个元素
        public void add(E e) {
            checkForComodification();
            try {
                int i = cursor;
                AbstractList.this.add(i, e);
                lastRet = -1;
                cursor = i + 1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
```
