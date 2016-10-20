#java Collection框架分析之HashSet实现简述#
---

##一、HashSet概述##

>This class implements the Set interface, backed by a hash table (actually a HashMap instance). It makes no guarantees as to the iteration order of the set; in particular, it does not guarantee that the order will remain constant over time. This class permits the null element.

HashSet使用HashMap实例保存数据并且实现了Set接口，继承自AbstractSet其实现了部分Set方法。HashSet不保证数据的遍历顺序，同时它也不保证顺序每次都是相同的，对于数据的存储HashSet允许null数据，类的声明如下

public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable

##二、实现分析##

+ 数据存储：HashSet使用HashMap(一般来说)作为数据存储

+ 添加元素：

```
	
	//使用要添加的元素作为HashMap的键从而保证元素不会重复
	public boolean add(E e) {
        return map.put(e, PRESENT)==null;//PRESET为假数据
    }

```

+ 删除元素和conatains判断:直接使用HashMap的方法

+ 遍历元素-Iterator；同样是快速失败的迭代器

```

	public Iterator<E> iterator() {
        return map.keySet().iterator();//不保证顺序
    }

```

##三、与LinkedHashSet的比较##

与HashSet相比，LinkedHashSet继承自HashSet,但是与之相反是它使用的是HashSet的另一个构造方法，该构造方法将HashSet存储数据的实例改为使用继承自HashMap的LinkedHashMap从而保证每个遍历时元素的顺序为插入时的位置，除非重新插入数据才会改变该数据在集合中的位置，此外二者都不保证多线程环境下的正确性