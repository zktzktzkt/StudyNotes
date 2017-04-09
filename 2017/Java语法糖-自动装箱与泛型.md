# Java语法糖-自动装箱与泛型
## 前言

如标题,在日常编程中自动装箱、泛型可以说是我们在程序中使用非常频繁地特性，但是这些特性都被称之为语法糖,也就是说这些特性其实并没有什么语言底层上的
支持或者改变什么的，只是一些简单的语法支持,帮助我们提高效率,所以说他们应该基于已有功能的一种封装,这里我们通过javap这个工具来反编译字节码来一窥这些特性的实现原理.

## 1.自动装箱与拆箱

通过下面这个简单的例子的分析
```
public class Main{
    public static void main(String[] args) {
       Integer a=200;
       int b=10;
       b=a;
       a=b;
    }
}
```
上面这里例子包含了变量Integer a初始化的自动装箱的情况、变量基本类型b的初始化、以及两种赋值的情况.单纯的从java代码上我们几乎无法知道这里面究竟做了什么来完成初始化的动作,毕竟**这些变量没有像其他的对象一样必须显示的使用了new来完成初始化**.所以接下来通过class字节码来观察这些操作是如何进行的.

首先通过javac、javap这两个命令来查看字节码

```
javac Main.java//编译.java文件
javap -v -p -c -s Main//使用javap反汇编.class文件
```
获得的字节码部分结果如下

```
Classfile /D:/Main.class
  Last modified 2017-4-9; size 384 bytes
  MD5 checksum 45d47c5af5f867efa24d7e5d19432431
  Compiled from "Main.java"
public class Main
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #5.#14         // java/lang/Object."<init>":()V
   #2 = Methodref          #15.#16        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #3 = Methodref          #15.#17        // java/lang/Integer.intValue:()I
   #4 = Class              #18            // Main
   #5 = Class              #19            // java/lang/Object
   ...
{
  public Main();
  ...
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: sipush        200 //将常量200送到栈顶
         3: invokestatic  #2 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;调用#2指向的静态方法valueOf
         6: astore_1        //将栈顶引用型数值存入第二个本地变量
         7: bipush        10 //将常量10送到栈顶
         9: istore_2        //将栈顶引用型数值存入第三个本地变量
        10: aload_1         //将第二个引用类型变量送到栈顶
        11: invokevirtual #3 // Method java/lang/Integer.intValue:()I 调用#3指向的静态方法initValue
        14: istore_2        //将栈顶引用型数值存入第三个本地变量
        15: iload_2         //将第三个引用类型变量送到栈顶
        16: invokestatic  #2// Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;调用#2指向的静态方法valueOf
        19: astore_1        //将栈顶引用型数值存入第二个本地变量
        20: return
    ...
}
```

由上面的字节码以及注释可得如下结果

```
public class Main{
    public static void main(String[] args) {
       Integer a=200;//这里实质调用了Integer的valueOf方法完成初始化，由字节码0、3、6完成,
       int b=10;//常量赋值,由字节码7、9完成
       b=a;//调用Integer的initValue直接将Integer的int值取出进行赋值,由字节码11、14、完成
       a=b;//同a=200,由字节码15、16、19完成
    }
}
```

从上面注释可以清楚的看到自动装箱、拆箱是通过调用valueOf、initValue进行的赋值.

## 2.泛型擦除

同样是用一个很简单的例子,以免得到的字码过长
```
public class Main {
    public static void main(String[] args) {
       List<String>  test = new ArrayList<>();
       test.add("test");
    }
}
```

同样使用javac、javap获得部分字节码如下

```
Classfile /D:/Main.class
  Last modified 2017-4-9; size 378 bytes
  MD5 checksum 4d7b865e1dd2c94c510fc00bc8f67261
  Compiled from "Main.java"
public class Main
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#16         // java/lang/Object."<init>":()V
   #2 = Class              #17            // java/util/ArrayList
   #3 = Methodref          #2.#16         // java/util/ArrayList."<init>":()V
   #4 = String             #18            // test
   #5 = InterfaceMethodref #19.#20        // java/util/List.add:(Ljava/lang/Object;)Z
   #6 = Class              #21            // Main
   #7 = Class              #22            // java/lang/Object
   .....
  #25 = Utf8               (Ljava/lang/Object;)Z
{
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2 //创建对象并将引用值压入栈顶
         3: dup              //复制栈顶并压入栈顶
         4: invokespecial #3 //执行构造方法
         7: astore_1         //存储到第二个本地变量
         8: aload_1          //将ArrayList实例对象推向栈顶
         9: ldc           #4 //将#4指向的常量test推向栈顶
        11: invokeinterface #5,  2// 执行add方法，可以看到#5指向的方法的方法签名为Object而不是String,其中2指得是参数个数
        16: pop
        17: return
     ...
}
```

所以虽然我们在java代码中使用<>来指定了泛型对象的类型，但是编译之后对象的类型实际上被转化为Object类型，而不是我们所声明的类型,这也就是为什么java
实现的是伪泛型.

## 3.foreach

这里只举一个foreach对于List的实现，对于数组的实现(类似for(int i=0;i<length;i++))可自行查看

```
public class Main {
    public static void main(String[] args) {
       List<String> arrs = new ArrayList<>();
       arrs.add("e");
       for(String i:arrs){
           i+=" loop";
       }
    }
}
```

字节码的部分实现如下

```
Classfile /D:/Main.class
  Last modified 2017-4-9; size 792 bytes
  MD5 checksum 3fc74e184cec4d0963d2cf05b2ed33ac
  Compiled from "Main.java"
public class Main
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #16.#28        // java/lang/Object."<init>":()V
   #2 = Class              #29            // java/util/ArrayList
   #3 = Methodref          #2.#28         // java/util/ArrayList."<init>":()V
   #4 = String             #30            // e
   #5 = InterfaceMethodref #31.#32        // java/util/List.add:(Ljava/lang/Object;)Z
   #6 = InterfaceMethodref #31.#33        // java/util/List.iterator:()Ljava/util/Iterator;
   #7 = InterfaceMethodref #34.#35        // java/util/Iterator.hasNext:()Z
   #8 = InterfaceMethodref #34.#36        // java/util/Iterator.next:()Ljava/lang/Object;
   #9 = Class              #37            // java/lang/String
  #10 = Class              #38            // java/lang/StringBuilder
  #11 = Methodref          #10.#28        // java/lang/StringBuilder."<init>":()V
  #12 = Methodref          #10.#39        // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #13 = String             #40            //  loop
  #14 = Methodref          #10.#41        // java/lang/StringBuilder.toString:()Ljava/lang/String;
  ...
{
  public Main();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #2  
         3: dup
         4: invokespecial #3   
         7: astore_1
         8: aload_1
         9: ldc           #4       
        11: invokeinterface #5,  2 //至此完成创建ArrayList并添加test到集合的过程
        16: pop                    //将栈顶数值弹出
        17: aload_1                //推到栈顶
        18: invokeinterface #6,  1 //执行List的iterator方法,参见常量池中的#6项
        23: astore_2               //将栈顶类型存入变量
        24: aload_2                //将变量推到栈顶
        25: invokeinterface #7,  1 //执行Iterator.hasNext()方法,参见常量池中的#7项
        30: ifeq          66       //栈顶类型等于0,跳转到66
        33: aload_2                //将变量推到栈顶
        34: invokeinterface #8,  1 //执行Iterator.next:()方法,参见常量池中的#8项
        39: checkcast     #9       //类型转化检查
        42: astore_3               //将栈顶类型存入变量
        43: new           #10      //创建 java/lang/StringBuilder对象
        46: dup                    //复制栈顶数值并压入栈顶
        47: invokespecial #11      //执行构造函数
        50: aload_3                //将变量推到栈顶
        51: invokevirtual #12      //调用append函数,append 变量i
        54: ldc           #13      //将#13指向的常量loop推向栈顶
        56: invokevirtual #12      //append 常量loop
        59: invokevirtual #14      //执行toString方法
        62: astore_3               //将栈顶类型存入变量
        63: goto          24       //调到24继续循环
        66: return
    ....
}
```

通过对比字节码命令的含义及反汇编得到的字节码，我们可以清楚的看到foreach这颗语法糖可以说就像是对用iterator进行遍历的函数封装，同时另一个发现是我们在java代码中使用**+**进行字符串的拼接实质上是会自动的转为StringBuilder进行操作.