---
layout: post
title:  "Java集合（四）：HashMap与HashSet"
date:   2018-11-21
desc: "Java集合（四）：HashMap与HashSet，首先将高16位无符号右移16位与低十六位做异或运算。如果不这样做，而是直接做&运算那么高十六位所代表的部分特征就可能被丢失 将高十六位无符号右移之后与低十六位做异或运算使得高十六位的特征与低十六位的特征进行了混合得到的新的数值中就高位与低位的信息都被保留了 ，而在这里采用异或运算而不采用& ，| 运算的原因是 异或运算能更好的保留各部分的特征，如果采用&运算计算出来的值会向1靠拢，采用|运算计算出来的值会向0靠拢"
keywords: "Java,Collection,HashMap,HashSet"
categories: [Java]
tags: [Java,Collection]
icon: icon-html
---
## 特点
- hash算法将高16位与低16位做异或操作获取hash值。
- 当单一链大于等于8时同时桶大小大于等于64时，会对节点链进行树化操作。
- 当扩容时，桶中元素个数小于6，就会把树形的桶元素 还原（切分）为链表结构
- 当键值对大小大于桶大小乘负载因子时，进行桶扩建操作
- 允许空键值
- 桶的大小为2的幂级数

## 基本操作
### Hash算法
首先将高16位无符号右移16位与低十六位做异或运算。如果不这样做，而是直接做&运算那么高十六位所代表的部分特征就可能被丢失 将高十六位无符号右移之后与低十六位做异或运算使得高十六位的特征与低十六位的特征进行了混合得到的新的数值中就高位与低位的信息都被保留了 ，而在这里采用异或运算而不采用& ，| 运算的原因是 异或运算能更好的保留各部分的特征，如果采用&运算计算出来的值会向1靠拢，采用|运算计算出来的值会向0靠拢
### 内部结构
``` java
//哈希桶，不主动进行初始化，空间不足的时候会进行扩充，大小为2的n次方
Node<K,V>[] table;
//键值对的个数
int size;
//集合结构化修改的次数
int modCount;
//阈值，当size>threshold时考虑进行拓展，其大小等于table.length*loadFactor
int threshold;
//负载因子，表示整体上table被占用的程度，默认为0.75
final float loadFactor;

Set<Map.Entry<K,V>> entrySet;
```
### 添加元素
<img src="{{ site.img_path }}/2018-12-05-1.png" width="75%">

``` java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) 
        //当table未初始化或者table为空时，进行初始化
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        //table[hash]这个桶位对应的槽为空时，直接将新节点放到这个槽里
        tab[i] = newNode(hash, key, value, null);
    else {
        //获取当前桶槽对应的与Key相同的节点
        Node<K,V> e; K k;
        if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
            //找到了这个key
            e = p;
        else if (p instanceof TreeNode)
            //没找到这个key，当时当前节点为treeNode，进行红黑树put操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //进行链表迭代，如果迭代到最后也没找到，创建一个新节点，如果这个链表过长，进行树化操作
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到了这个key，直接跳出
                if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果当前map存在这个key，修改value值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果键值对大小大于阈值，进行桶扩建
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 空间扩充
``` java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //桶原来的大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) { //桶的大小不为空
        if (oldCap >= MAXIMUM_CAPACITY) { //桶已经扩充到最大了，无法继续扩建了
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 还能扩充则阈值与新桶大小变成2倍
    }else if (oldThr > 0){ // 阈值不为零，同时桶的大小为0，将新桶大小设置为原来的阈值
        newCap = oldThr;
    }else {               // 都为零，第一次初始化
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) { //新阈值为零的话，设置一个阈值为newCap*loadFactor
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?(int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"}) //申请新空间
       Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) { //当旧桶有数据时，进行数据复制
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null){newTab[e.hash & (newCap - 1)] = e;} //这个槽只有一个节点
                else if (e instanceof TreeNode) //这个槽只有一个树节点
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); //拆分红黑树节点
                else { // 这个槽有一个链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) { //位置不用动
                            if (loTail == null){loHead = e;}
                            else{loTail.next = e;}
                            loTail = e;
                        }
                        else { //位置要移动到当前位置+oldCap
                            if (hiTail == null){hiHead = e;}
                            else{hiTail.next = e;}
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
### 获取元素
获取元素的逻辑和插入元素其实相似，在这里不再进行分析
``` java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
### 删除元素
``` java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 && //在桶的对应槽里找到了元素
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) //如果第一个节点的key就匹配了
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode) //红黑树节点的处理
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null); //循环查找
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) //如果是树节点，就从树中移除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) //链表头是匹配的节点
                tab[index] = node.next;
            else //链表头不是匹配的节点
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
## HashMap在1.8中树化的操作
### 桶的树形化 treeifyBin()
``` java
//将桶内所有的 链表节点 替换成 红黑树节点
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果当前哈希表为空，或者哈希表中元素的个数小于 进行树形化的阈值(默认为 64)，就去新建/扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //如果哈希表中的元素个数超过了 树形化阈值，进行树形化
        // e 是哈希表中指定位置桶里的链表节点，从第一个开始
        TreeNode<K,V> hd = null, tl = null; //红黑树的头、尾节点
        do {
            //新建一个树形节点，内容和当前链表节点 e 一致
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null) //确定树头节点
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);  
        //让桶的第一个元素指向新建的红黑树头结点，以后这个桶里的元素就是红黑树而不是链表了
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}


    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}

final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) { //头回进入循环，确定头结点，为黑色
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {  //后面进入循环走的逻辑，x 指向树中的某个节点
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            //又一个循环，从根节点开始，遍历所有节点跟当前节点 x 比较，调整位置，有点像冒泡排序
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;        //这个 dir 
                K pk = p.key;
                if ((ph = p.hash) > h)  //当比较节点的哈希值比 x 大时， dir 为 -1
                    dir = -1;
                else if (ph < h)  //哈希值比 x 小时 dir 为 1
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 如果比较节点的哈希值、 x 
                    dir = tieBreakOrder(k, pk);
                    //把 当前节点变成 x 的父亲
                    //如果当前比较节点的哈希值比 x 大，x 就是左孩子，否则 x 是右孩子 
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```
### 红黑树中添加元素 putTreeVal()
``` java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                   int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        TreeNode<K,V> root = (parent != null) ? root() : this;
        //每次添加元素时，从根节点遍历，对比哈希值
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))  
            //如果当前节点的哈希值、键和要添加的都一致，就返回当前节点（奇怪，不对比值吗？）
                return p;
            else if ((kc == null &&
                      (kc = comparableClassFor(k)) == null) ||
                     (dir = compareComparables(kc, k, pk)) == 0) {
                //如果当前节点和要添加的节点哈希值相等，但是两个节点的键不是一个类，只好去挨个对比左右孩子 
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null &&
                         (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                         (q = ch.find(h, k, kc)) != null))
                        //如果从 ch 所在子树中可以找到要添加的节点，就直接返回
                        return q;
                }
                //哈希值相等，但键无法比较，只好通过特殊的方法给个结果
                dir = tieBreakOrder(k, pk);
            }

            //经过前面的计算，得到了当前节点和要插入节点的一个大小关系
            //要插入的节点比当前节点小就插到左子树，大就插到右子树
            TreeNode<K,V> xp = p;
         //这里有个判断，如果当前节点还没有左孩子或者右孩子时才能插入，否则就进入下一轮循环 
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                Node<K,V> xpn = xp.next;
                TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                xp.next = x;
                x.parent = x.prev = xp;
                if (xpn != null)
                    ((TreeNode<K,V>)xpn).prev = x;
                //红黑树中，插入元素后必要的平衡调整操作
                moveRootToFront(tab, balanceInsertion(root, x));
                return null;
            }
        }
    }

    //这个方法用于 a 和 b 哈希值相同但是无法比较时，直接根据两个引用的地址进行比较
    //这里源码注释也说了，这个树里不要求完全有序，只要插入时使用相同的规则保持平衡即可
     static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
             compareTo(b.getClass().getName())) == 0)
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                 -1 : 1);
        return d;
    }
```
### 红黑树中查找元素 getTreeNode()
``` java
final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);
}
    //从根节点根据 哈希值和 key 进行查找
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        TreeNode<K,V> p = this;
        do {
            int ph, dir; K pk;
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p;
            else if (pl == null)
                p = pr;
            else if (pr == null)
                p = pl;
            else if ((kc != null ||
                      (kc = comparableClassFor(k)) != null) &&
                     (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.find(h, k, kc)) != null)
                return q;
            else
                p = pl;
        } while (p != null);
        return null;
    } 
```
### 树形结构修剪 split()
``` java
//参数介绍
    //tab 表示保存桶头结点的哈希表
    //index 表示从哪个位置开始修剪
    //bit 要修剪的位数（哈希值）
    final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        TreeNode<K,V> b = this;
        // Relink into lo and hi lists, preserving order
        TreeNode<K,V> loHead = null, loTail = null;
        TreeNode<K,V> hiHead = null, hiTail = null;
        int lc = 0, hc = 0;
        for (TreeNode<K,V> e = b, next; e != null; e = next) {
            next = (TreeNode<K,V>)e.next;
            e.next = null;
            //如果当前节点哈希值的最后一位等于要修剪的 bit 值
            if ((e.hash & bit) == 0) {
                    //就把当前节点放到 lXXX 树中
                if ((e.prev = loTail) == null)
                    loHead = e;
                else
                    loTail.next = e;
                //然后 loTail 记录 e
                loTail = e;
                //记录 lXXX 树的节点数量
                ++lc;
            }
            else {  //如果当前节点哈希值最后一位不是要修剪的
                    //就把当前节点放到 hXXX 树中
                if ((e.prev = hiTail) == null)
                    hiHead = e;
                else
                    hiTail.next = e;
                hiTail = e;
                //记录 hXXX 树的节点数量
                ++hc;
            }
        }


        if (loHead != null) {
            //如果 lXXX 树的数量小于 6，就把 lXXX 树的枝枝叶叶都置为空，变成一个单节点
            //然后让这个桶中，要还原索引位置开始往后的结点都变成还原成链表的 lXXX 节点
            //这一段元素以后就是一个链表结构
            if (lc <= UNTREEIFY_THRESHOLD)
                tab[index] = loHead.untreeify(map);
            else {
            //否则让索引位置的结点指向 lXXX 树，这个树被修剪过，元素少了
                tab[index] = loHead;
                if (hiHead != null) // (else is already treeified)
                    loHead.treeify(tab);
            }
        }
        if (hiHead != null) {
            //同理，让 指定位置 index + bit 之后的元素
            //指向 hXXX 还原成链表或者修剪过的树
            if (hc <= UNTREEIFY_THRESHOLD)
                tab[index + bit] = hiHead.untreeify(map);
            else {
                tab[index + bit] = hiHead;
                if (loHead != null)
                    hiHead.treeify(tab);
            }
        }
    }
```

## HashSet
HashSet的底层持有了一个HashMap对象，一切操作都通过HashMap进行 所以不需要详细讲解