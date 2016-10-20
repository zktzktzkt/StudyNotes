#java Collection框架分析之ArrayList实现分析#
----

##一、ArrayList概述##
>Resizable-array implementation of the List interface

如文档所说ArrayList是一种基于List接口的可变长数组的实现，类的声明如下

>public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable

##二、问题分析##

###1、ArrayList的数据结构###

ArrayList属于线性表的一种，一般来说线性表有两种实现方式：

一种是以数组形式的连续内存的实现，这种方式可以快速的访问数据但是删除之类的操作不灵活，另一种是以链表形式的非连续内存的实现。ArrayList采用的是前者的方式，所以显然ArrayList的数据使用的数组的形式来保存的.ArrayList中声明的几个变量如下

```

	private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData;

```

可以看到ArrayList声明的变量只有三个数组，两个静态空数组，一个不需要序列化的数组。很显然两个空数组不会是用来存储数据，只有elementData是用来存储数据的，通过注释以及后面add等方法的逻辑，我们可以知道的是这两个空元素是但ArrayList为空时，elementData的默认引用，**只所以存在两个是由于根据elementData引用的不同的空实例，但我们添加第一个元素时，elementData变量初始化会存在差异**,ArrayList在构造函数中对elementData的初始化逻辑如下

```
	//指定初始化控件大小为空时，elementData指向EMPTY_ELEMENTDATA
	public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        }
    }
    //传入的集合为空时同上elementData指向EMPTY_ELEMENTDATA
    public ArrayList(Collection<? extends E> c) {
        //...
    }
    //默认构造函数elementData指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

```

###2、添加、查找、删除元素的实现###

+ 添加的实现add系列方法：两个add方法，两个addAll方法

```

	public boolean add(E e) {
		//传入ensureCapacityInternal的参数是需要的大小
        ensureCapacityInternal(size + 1);//添加元素前确定当前数组大小是否足够，不够会自动扩大，下面ArrayList的内存增长时分析
        elementData[size++] = e;//保存元素
        return true;
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);//检查下表是否超出边界
        ensureCapacityInternal(size + 1);  //确认数组大小，同上
        //调用系统的copy方法将elementData的元素从index开始整个后移，将elemenData从index开始的元素复制到index+1开始的位置
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        elementData[index] = element;//保存元素
        size++;
    }
    //addAll(Collection<? extends E> c)逻辑与之类似，只不过少了移动elementData，直接将数据插到末尾
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);//检查边界
        Object[] a = c.toArray();//转化为数据
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // 确认空间
        int numMoved = size - index;
        if (numMoved > 0)//移动elementData的数据为a数组腾出空间
            System.arraycopy(elementData, index, elementData, index + numNew,numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);//拷贝数组
        size += numNew;//增加元素个数
        return numNew != 0;
    }


```

从上面的三个方法的代码能够直观的看到ArrayList添加元素的逻辑:(检查边界)->确认数组空间->(移动elementData数据)->拷贝数据到elementData中

+ 查找方法indexOf的实现:

```
	
	public int indexOf(Object o) {
        if (o == null) {//ArrayList支持空引用的存在，对于空引用同样查找
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {//遍历通过equals判断
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

```

ArrayList通过简单的遍历数组元素比较的形式来查找引用是否存在，lastIndexOf这是通过从size-1进行查找来达到反向查找的目的，同时ArrayList的contains方法也是通过调用indexOf方法根据其返回值来判断是否存在元素

+ 删除元素remove系列方法的实现

```
	//removeRange方法与之类似
	public E remove(int index) {
        rangeCheck(index);
        modCount++;
        E oldValue = elementData(index);
        int numMoved = size - index - 1;
        if (numMoved > 0)//如果不是从末尾删除数据则需要移动elementData的数据
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        elementData[--size] = null; // 减少size长度，元素置空
        return oldValue;
    }

    //只删除查找到的第一个元素，没有这不改变
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);//删除查找到元素的索引使用fastRemove，速度较快
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    //相较remove方法不用检查边界，速度快一点
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

```

删除vhude逻辑同样很简单：检查边界或者查找元素->elementData数据移动->待删除的元素索引置空即可

###3、ArrayList元素的遍历-iterator方法###

这里子分析Iterator迭代器方法的实现，即迭代模式,ArrayList中与迭代器相关的方法一共有三个iterator、listIterator、listIterator(int index)，后者的实现与Iterator类似，只看iterator方法的实现如下

```

	public Iterator<E> iterator() {
        return new Itr();//直接返回内部类Itr迭代器类实例
    }

    //Itr类的部分代码如下
	private class Itr implements Iterator<E> {
        int cursor;       // 表示当前遍历的元素的位置
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;//初始化时保存当前List修改的次数

        public boolean hasNext() {
            return cursor != size;
        }
        public E next() {
            checkForComodification();//检查是否修改过
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;//直接引用ArrayList实例的elementData数据元素
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];//返回元素，并修改lastRet的值
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();//检查修改
            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;//删除元素后当前的cursor值变为lastRet，即上一次检索的索引值
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
        //对每个元素执行consumer的操作
        public void forEachRemaining(Consumer<? super E> consumer) {
            
        }

        final void checkForComodification() {//检查当前的修改次数与创建Ite时修改次数是否相同
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```

从Ite的实现来看这个类的方法满足文档所述的ArrayList的迭代器是快速失败的，我们在使用Iterator迭代器遍历目标时不能使用ArrayList类来修改元素

###4、内存操作###

+ 先看每次添加元素内存检查逻辑

```
	//minCapacity参数表示当前需要的内存大小
	private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

```

在上面的ensureCapacityInternal方法中可以看到DEFAULTCAPACITY_EMPTY_ELEMENTDATA的作用，即对于ArrayList的两个空元素来说，当我们第一次添加元素时如果elementData的空为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则ArrayList在初始化elementData时会**将DEFAULT_CAPACITY参与初始化时空间**的比较,接着ensureExplicitCapacity代码如下

```

	private void ensureExplicitCapacity(int minCapacity) {
        modCount++;//修改次数增加
        if (minCapacity - elementData.length > 0)//需要的内存大于当前数组的空间
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);//扩大1.5倍
        if (newCapacity - minCapacity < 0)//如果小于需要的空间
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);//检查minCapacity是否溢出限制newCapacity不超过最大内存
        elementData = Arrays.copyOf(elementData, newCapacity);//最终调用System.arraycopy复制指定长度的数组，完成空间扩充
    }

```

此外还有与之类似的ensureCapacity方法，与之相反的是trimToSize方法是用缩减内存的

```
	//将ArrayList的无效内存删除
	public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)? EMPTY_ELEMENTDATA: Arrays.copyOf(elementData, size);
        }
    }

```

###5、ArrayList的clone###

```

	public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);//拷贝关键数据
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }

```

###6、序列化操作###

```

	private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        int expectedModCount = modCount;
        s.defaultWriteObject();
        s.writeInt(size);
        //虽然元素声明为transient但是这里手动写入了
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
    //按write序读出数据
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        s.defaultReadObject();
        s.readInt(); // ignored
        if (size > 0) {
            ensureCapacityInternal(size);
            Object[] a = elementData;
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }

```


##三、与Vector相比的异同##

相同点:

+ 都是线性表的数组形式的实现
+ 对于增加、删除、查找等操作逻辑类似
+ 内存增长策略基本相同
+ 迭代器都是快速失败的

不同点:

+ 初始化时内存的策略不同，Vector一开始便分配内存
+ Vector的大部分方法带有synchronized,Vector是在多线程环境下是线程安全的
+ Vector存在Enumeration实现的非快速失败的遍历方法

