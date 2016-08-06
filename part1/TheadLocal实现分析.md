##TheadLocal源码分析
**一、ThreadLocal是做什么用的?**

先从文档开始看：

>Implements a thread-local storage, that is, a variable for which each thread has its own value. All threads share the same ThreadLocal object, but each sees a different value when accessing it, and changes made by one thread do not affect the other threads. The implementation supports null values.

大意是说它实现了一种存储方式，使得每一个线程共享一个ThreadLocal对象，但是对于每一个线程对于ThreadLocal对象中值的修改不影响其他线程的ThreadLocal的值，每个线程获取的值可以是不一样的。

**二、ThreadLocal实现方法的分析**

以下源码为jdk1.8

从文档可以看出来ThreadLocal主要包括set、get方法，以下也是主要关注这两个的实现

首先看这set方法的实现
```
    public void set(T value) {
        Thread t = Thread.currentThread();
        //返回Thread对象下的threadLocals字段，且从Thread类中关于该字段的代码及文档该字段的管理是由ThreadLocal类来做，
        //Thread本身不对其做任何操作，只在线程的exit方法中将其置为空
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);//不为空，则修改副本的值
        else
            createMap(t, value);//否则创建ThreadLocalMap并设置副本的值
    }

```
可以看出来其核心是一个ThreadLocalMap类，文档中对这个类的解释是这个类是一个定制化的HashMap，专门由于维护thread local values，接着看调用
```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);//计算索引值
            table[i] = new Entry(firstKey, firstValue);//创建副本来备份初值
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            //使用threadLocalHashCode获取索引值，使得对于不同的ThreadLocal有不同的副本
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {//遍历Entry,匹配key则设置值
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);//没有相应的Entry则创建并赋值
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
上面的代码虽然有点长，但是由文档的HashMap提示可以看到，主要的逻辑就是根据threadLocalHashCode这个字段和Entry数组的长度来计算值的索引，并且最终这个值的备份存在Entry数组中

有了上面的分析，可以猜想ThreadLocal的get方法就是获取ThreadLocal对象，然后获取Entry中对应的数据项

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();//设置初始值
    }

    private T setInitialValue() {
        T value = initialValue();//调用初始化值方法
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
从代码来看get方法首先先利用ThreadLocalMap来获取Entry的数据，如果只为空则调用初始化方法initialValue并将该值的备份赋到Entry中。这也就是为什么使用ThreadLocal的时候一般会重写initialValue来返回初值

**三、总结**

ThreadLocal使用initialValue这个方法为每个线程提供初始值或者直接使用set方法设置值，而实现各个线程对共享值得操作而不互相影响的原理则是利用了**Thread类的中的threadLocals字段来存储副本(该类是一个专门为ThreadLocal实现的一个定制化hashmap)，所以说ThreadLocal实际上是让Thread类自身来存储共享变量的副本，并对副本做操作从而达到互不影响的效果**