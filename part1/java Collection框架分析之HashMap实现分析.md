#java Collection框架分析之HashMap实现分析#
---

##一、HashMap概述##

HashMap是实现了Map接口的哈希表，它与JDK中的HashTable基本相同，但是**HashMap不是同步的**且HashMap允许空key、value，此外对于get、put等基本操作HashMap保证常数级的时间，影响其性能的主要参数为其初始化时的容量和增长因子

##二、实现分析##

###1、HashMap的数据结构###

```

	transient Node<K,V>[] table;//存储数据的变量，其大小会根据需要调整，第一次使用时初始化
    transient Set<Map.Entry<K,V>> entrySet;// Note that AbstractMap fields are used for keySet() and values().
    transient volatile Set<K>        keySet;//key集合
    transient volatile Collection<V> values;//值集合

```

###2、添加、查找、删除元素的实现###

+ 添加键值对put系列方法：

```
	//如果key相同，新值会替代旧值
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    //onlyIfAbsent参数指示如果元素存在则不要覆盖
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
        Node<K,V>[] tab; 
        Node<K,V> p; 
        int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)//但数组table为空或者长度为0，进行大小调整
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)//如果当前key的hash值计算的数组下标处元素为空，则当前元素成为这个链表的第一个元素
            tab[i] = newNode(hash, key, value, null);//创建数据节点
        else {
            Node<K,V> e; 
            K k;
            //如果hash值等于表头的hash值且键值引用也相等  或者 键值的内容相等
            if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)//如果链表已经修改为红黑树的结构，则调用红黑树的插入方法插入
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {//默认的冲突情况，遍历链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {//到达链表末尾
                        p.next = newNode(hash, key, value, null);//元素插入
                        if (binCount >= TREEIFY_THRESHOLD - 1) // 如果链表长度达到转化为红黑树的触发值
                            treeifyBin(tab, hash);
                        break;
                    }
                    //数据已存在
                    if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // 数据存在根据onlyIfAbsent参数决定是否需要替换
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);//LinkedHashMap有效，其他为空实现
                return oldValue;//返回旧值
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);//LinkedHashMap有效，其他为空实现
        return null;//新插入元素返回值为null
    }

```

+ 通过Key获取数据get方法

```

	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    //通过Hash码和key来定位数组
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; 
        Node<K,V> first, e; 
        int n; 
        K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&(first = tab[(n - 1) & hash]) != null) {//列表不空，且hash码所指链表存在元素
            if (first.hash == hash &&((k = first.key) == key || (key != null && key.equals(k))))//首先检查第一个元素是否命中
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)//红黑树结构的查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {//链表形式直接遍历查找
                    if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

+ 遍历方法，文档中提到Map提供了三种集合以供使用:key-键值集合、values-值集合、entrySet-键值对集合

```

    public Set<K> keySet() {
        Set<K> ks;
        return (ks = keySet) == null ? (keySet = new KeySet()) : ks;
    }

    //keySet的iterator接口的iterator方法如下实现
    public final Iterator<K> iterator() { 
        return new KeyIterator();
    }
    final class KeyIterator extends HashIterator implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }
    //HashIterator寻找下一个key的方法如下实现
    final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {//当前桶遍历结束时
                do {} while (index < t.length && (next = t[index++]) == null);//寻找下一个桶的key
            }
            return e;//返回的是键対值的节点
    }

```

由于HashIterator返回的是一个key-value的键対值的Node节点，所以values() 、entrySet()方法同样也利用了该类来实现遍历，只不过封装类稍有不同 

###4、内存操作###

先看成员变量threshold(下一次需要重新分配内存的锚值)的初始化

```
	
	//用计算距离cap最近的大于cap的2的幂作为值返回给threshold
	static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

再看HashMap的reSize过程，部分代码如下

```

	final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//取当前的容量
        int oldThr = threshold;//取当前的扩充容量的门槛
        int newCap, newThr = 0;
        //计算新的容量和门槛.....略
        
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//重新创建一个新大小的数组
        table = newTab;
        if (oldTab != null) {//复制数据到新数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)//只有头节点的情况
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//调整红黑树，会根据红黑树节点的个数来决定是否需要退化红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //这里一个重要的点就是根据HashMap的策略，部分元素需要重新安排其在数组中的位置，使之能够满足新的
                    //hash映射条件newTab[e.hash & (newCap - 1)] = e;
                    else { 
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
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
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;//调整位置
                        }
                    }
                }
            }
        }
        return newTab;
    }

```

分配的逻辑很显然，受限计算新的容量和门槛，然后将原先的数据复制到新的数组中。新的容量和门槛计算分以下几种情况

+ 当现在的容量和门槛都为0时，新容量为默认值DEFAULT_INITIAL_CAPACITY、(int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
+ 当现在的容量为0，门槛>0时，新容量为当前门槛，新门槛为新容量乘以增长因子或者最大值Integer.MAX_VALUE
+ 当现在的容量>0时,新容量为当前容量的两倍，新门槛为当前门槛的两倍或者新容量乘以增长因子或者最大值Integer.MAX_VALUE

然后就是将数据拷贝到新数组中，这里的一个要点就是要使得每个元素的hash值和新的容量计算下标时能够正确命中。

###5、从链表转向红黑树的时机###

首先HashMap中声明了两个常量

```

    static final int TREEIFY_THRESHOLD = 8;//将链表转化为红黑树的门槛

    static final int UNTREEIFY_THRESHOLD = 6;//将红黑树退化为链表的门槛

```

然后就在putVal方法中，在想链表插入元素时判断链表长度决定是否转化为红黑树的形式,该段代码如下

```

    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) 
            treeifyBin(tab, hash);//转化为红黑树
            break;
    }

    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; 
        Node<K,V> e;
        //数组长度太小则调整数组大小来避免冲突
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);//将该链表上的键対值数据替换为TreeNode
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);//进行数据结构的调整改造
        }
    }

```

同时当红黑树节点过少时，在resize方法中会调用TreeNode的split的方式将红黑树退化为链表

###6、与LinkedHashMap的不同###

相较于子类LinkedHashMap，该类保证了每次遍历时能够得到确定的顺序，具体实现是修改了HashMap的每个键対值的数据结构，将其扩充为可用于链表的存储结构，即LinkedHashMap使用了一个双向链表存储了节点，在遍历数据时通过该数据结构来遍历顺序来保证得到确定的顺序，具体插入节点的代码如下

```
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

```

即LinkedHashMap通过重写了创建新节点的方法在创建新节点时将节点插入了链表中，所以**LinkedHashMap遍历的顺序是插入序**,除非节点删除了重新插入否则不会改变遍历顺序

##三、HashMap数据结构总结##

HashMap在运行过程中会使用根据数据节点的数量使用两种不同的方式来解决哈希冲突的问题，其数据结构的变化如下图

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/picture/hashmap_datastruct.png)