# java Collection框架分析之LinkedList实现分析
---

## 一、LinkedList概述
>Doubly-linked list implementation of the List and Deque interfaces

LinkedList是一种双向链表并且实现了List接口、Deque接口，所以它还可以作为队列来使用，类的声明如下

>public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable

## 二、实现分析

### 1、LinkedList的数据结构

与ArrayList相对，LinkedList使用**双向链表**而非数组来实现线性表的结构，其声明的数据变量以及变量的类如下

```

	transient Node<E> first;//指向第一个节点

    transient Node<E> last;//指向最后一个节点

    private static class Node<E> {
        E item;//数据域
        Node<E> next;//指向下一个元素的指针域
        Node<E> prev;//指向上一个元素的指针域

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

```

### 2、添加、查找、删除元素的实现

+ 添加的实现add系列方法:这里只看List接口的方法，而不看Deque接口的方法，队列相对而言只是受限的线性表,同样这里有两个add方法，两个addAll方法

```

	public boolean add(E e) {
        linkLast(e);//插入到最后一个节点(last)后面
        return true;
    }
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);//创建节点，并设置指针域
        last = newNode;
        if (l == null)//如果为空当前的新节点成为第一个节点
            first = newNode;
        else//否则插入到最后一个节点后面
            l.next = newNode;
        size++;
        modCount++;
    }
    //插入到指定位置,element成为index节点
    public void add(int index, E element) {
        checkPositionIndex(index);//边界检查，对比size是否越界
        if (index == size)//插入到最后一个节点与add逻辑相同
            linkLast(element);
        else
            linkBefore(element, node(index));//node(index)方法遍历查找节点
    }
    //index表示要查询的节点位置
    Node<E> node(int index) {
        if (index < (size >> 1)) {//判断位置是位于前半段还是后半段从而加快检索速度
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
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;//原节点前指针域修改
        if (pred == null)//插入到第一个节点
            first = newNode;
        else//链接指针域
            pred.next = newNode;
        size++;
        modCount++;
    }

```

与平常我们学到的双向链表实现没有太大区别，不过使用了两个指针域，这样在索引的时候可以加快速度，addAll方法与add相比只不过多了一个遍历集合元素然后一个个添加的过程

+ 查找方法indexOf的实现: 这个基本和ArrayList逻辑相同，遍历节点比较

+ 删除元素remove系列方法的实现

```

	public E remove(int index) {
        checkElementIndex(index);//检查边界
        return unlink(node(index));
    }

    E unlink(Node<E> x) {
        final E element = x.item;
        final Node<E> next = x.next;//取前后指针域
        final Node<E> prev = x.prev;
        if (prev == null) {//移除的为第一个节点则修改第一个节点为后指针域
            first = next;
        } else {//否则链接前后指针域
            prev.next = next;
            x.prev = null;
        }
        if (next == null) {//同上修改last指针域
            last = prev;
        } else {//链接节点
            next.prev = prev;
            x.next = null;
        }
        x.item = null;
        size--;
        modCount++;
        return element;//返回删除元素
    }

```

### 3、LinkedList元素的遍历-iterator方法

```
	
	//返回指定第一个遍历元素的Iterator
	public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);//检查边界
        return new ListItr(index);
    }
    //部分代码如下
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;
        private Node<E> next;
        private int nextIndex;
        private int expectedModCount = modCount;

        ListItr(int index) {
            next = (index == size) ? null : node(index);//获取当前需要遍历的锚点
            nextIndex = index;
        }


        public E next() {
            checkForComodification();//同ArrayList检查是否已修改过
            if (!hasNext())
                throw new NoSuchElementException();
            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();
            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```

可见LinkedList的迭代器与ArrayList的实现基本类似，同样是快速失败等特性，同时LinkedList还另您一个descendingIterator() 获取迭代器的方法，这个方法返回的迭代器利用ListItr类实现了JDK自带的标准Iterator接口，更加方便

最后关于LinkedList的clone方法与ArrayList实现大体相似，手动复制或者读写关键的数据，但是需要注意的是LinkedList的clone方法返回的新LinkedList中所引用的数据与原先相同，**如果修改新的LinkedList中元素的值原先集合中数据的值也会改变**，ArrayList的clone方法也存在和LinkedList的clone方法相同的问题
