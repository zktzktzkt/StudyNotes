# ClassLoader的自定义的几种姿势

## 1.ClassLoader概述
>A class loader is an object that is responsible for loading classes. The class ClassLoader is an abstract class. Given the binary name of a class, a class loader should attempt to locate or generate data that constitutes a definition for the class. A typical strategy is to transform the name into a file name and then read a "class file" of that name from a file system.

即如我们所熟知的,ClassLoader是一个负责加载类的对象,他根据类的全限定名的获取指定类的class信息,并将其加载到JVM中生成Class对象以供使用.它将类加载的过程做了良好的封装.但是当我们需要利用ClassLoader加载位于其他路径的类,如果从网络、文件系统时,就需要对ClassLoader进行自定义,类似JAVA中已有的几种类加载器根据类的限定名来从各自所辖区域的进行查找获取,我们在自定义ClassLoader中可以说几乎唯一的任务就是根据类的限定名找到其所在的位置,然后获取它的二进制流,然后什么的不用管交给系统自己去加载就好.java中的几种ClassLoader的类搜索路径如下

+ Bootstrap ClassLoader:加载JAVA_HOME路径下的\lib目录
+ Extension ClassLoader:加载JAVA_HOME路径下的\lib\ext目录
+ Application ClassLoader:加载用户类路径上所指定的类库ClassPath

接下主要分析Java以及Android中几个自定义的ClassLoader是如何实现类路径的查找、类的二进制流的读取.在介绍之前首先先简单的浏览一下文档中关于ClassLoader的几个API的解释

## 2.核心API的解释

### 2.1 loadClass方法
文档的解释比较简单,就是说这个方法是根据类的全限定名来加载class的,在ClassLoader中的实现如下

```
protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        //第一步首先检查是否已经加载，如果已经加载则直接返回
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                //根据双亲委派的定义,首先交由父ClassLoader进行加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
            }
            //父ClassLoader加载失败,此时尝试自己加载,通过调用findClass方法来寻找加载路径,获取class文件流
            if (c == null) {
                long t1 = System.nanoTime();
                c = findClass(name);
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 2.2 findClass方法
这个方法是根据类的全限定名来查找class的路径,默认的实现是抛出异常,如下

```
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
}
```

### 2.3 defineClass方法
将字节数组转化为class实例

```
protected final Class<?> defineClass(String name, byte[] b, int off, int len,ProtectionDomain protectionDomain)
throws ClassFormatError
{
    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);//native
    postDefineClass(c, protectionDomain);
    return c;
}
```

ok,下一步主要开始分析java、Android中的实现

## 3.URLClassLoader
URLClassLoader根据URL从 **JAR文件** 或者 **文件目录**中搜索路径加载类和资源.也就是说它能够将一个jar包或者目录作为整体加入到类的搜索路径中,然后从jar或者目录中搜索到制定的类文件然后得到类实例.下面主要看下它的关键方法类的搜索的实现.

### 3.1 findClass类搜索的实现

截取关键部分如下
```
protected Class<?> findClass(final String name) throws ClassNotFoundException
{
    final Class<?> result;
    String path = name.replace('.', '/').concat(".class");
    //ucp是一个URLClassPath实例,该类包含了获取class文件二进制流的两个加载器FileLoader、JarLoader从而提供了从文件系统和jar包中获取class
    //文件的二进制流的能力
    Resource res = ucp.getResource(path, false);
    if (res != null) {
    //如果拿到则调用defineClass产生class实例对象并返回
        try {
            return defineClass(name, res);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
        } else {
            return null;
        }
    return result;
}
```

### 4.BaseDexClassLoader
该类提供基于dex文件的基础服务,它的搜索代码很短，基本上都交由DexPathList这个处理
```
 protected Class<?> findClass(String name) throws ClassNotFoundException 
 {
   List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
   //直接由DexPathList处理,并且完成了搜索,到实例化class的过程,而DexPathList则直接将任务由委托到DexFile类,由dex文件完成class的加载
   Class c = pathList.findClass(name, suppressedExceptions);
   if (c == null) {
       ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
       for (Throwable t : suppressedExceptions) {
           cnfe.addSuppressed(t);
       }
       throw cnfe;
   }
   return c;
 }
```
### 5.PathClassLoader、DexClassLoader
+ PathClassLoader:Android用于从本地的文件系统中进行加载系统class和Application class,代码的实现只是简单的继承了BaseDexClassLoader
+ DexClassLoader:这个类用于加载jar包、apk包中的.dex文件实体,可以用于加载那些未安装到系统的apk的class文件,需要注意的是它需要一个程序私有且可写的目录来缓存优化后的class

### 6.Android 如何加载外部apk中的class

从上面的描述可以知道要想从外部apk中加载class,我们就要让Android中的ClassLoader能够找到我们的apk中的class，换言之我们要将外部apk加入到ClassLoader的搜索路径中.此外由于Anroid中的ClassLoader主要功能都由BaseDexClassLoader实现,那么我们可以从这个类中找到线索,代码中我们知道的是BaseDexClassLoader只是将这个任务转给了DexPathList这个类(这个类的文档解释是它包含了关于dex/resource、native代码的路径, **即搜索路径**),然后这个类从它的路径中findClass的实现交由每个Element中的DexFile实现,而DexFile类则代表dex文件.那么如果我们能够根据apk获取到dex文件,然后将它的路径加入到DexPathList中的Element数组中,那么系统就能自动的搜索到外部apk中的类,并将其加载.

所以思路就是:根据apk获取dex文件->获取DexFile对象->将DexFile对象加入到PathList中.

一个关于加载外部apk的Activity的例子[load_apk_demo](https://github.com/stdnull/demo/tree/master/load_apk_demo)

### 参考
+ [Android 插件化原理解析——插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)
