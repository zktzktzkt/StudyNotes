# 从字节码看类的初始化及finally执行顺序

## 1 类的初始化

为了生成的字节码尽量短小,故没有打印相关的值,测试代码如下

```
public class Father {
	int FATHER_NORMAL = 0;
	public static int FATHER_STATIC = 1;
	public static final int FATHER_STATIC_FINAL = 2;
	public final int FATHER_FINAL = 3;
	public Father(){
		int father = 4;
	}
}
```

得到的反编译代码注释如下

```
Classfile /D:/Father.class
  Last modified 2017-5-7; size 450 bytes
  MD5 checksum 47040b5c83d813ae4cf1a9472056fcf7
  Compiled from "Father.java"
public class main.Father
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#22         // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#23         // main/Father.FATHER_NORMAL:I
   #3 = Fieldref           #5.#24         // main/Father.FATHER_FINAL:I
   #4 = Fieldref           #5.#25         // main/Father.FATHER_STATIC:I
   #5 = Class              #26            // main/Father
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               FATHER_NORMAL
   #8 = Utf8               I
   #9 = Utf8               FATHER_STATIC
  #10 = Utf8               FATHER_STATIC_FINAL
  #11 = Utf8               ConstantValue 
  //...
{
  int FATHER_NORMAL;
    descriptor: I
    flags:

  public static int FATHER_STATIC;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC

  public static final int FATHER_STATIC_FINAL;//注意该成员的描述符
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 2

  public final int FATHER_FINAL;
    descriptor: I
    flags: ACC_PUBLIC, ACC_FINAL
    ConstantValue: int 3

  public main.Father();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=1
         0: aload_0
         1: invokespecial #1  //调用父类Object的构造方法 Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_0
         6: putfield      #2 //初始化变量 Field FATHER_NORMAL:I
         9: aload_0
        10: iconst_3
        11: putfield      #3 //初始化变量 Field FATHER_FINAL:I
        14: iconst_4
        15: istore_1
        16: return
        //...

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: iconst_1
         1: putstatic     #4  //初始化静态变量 Field FATHER_STATIC:I
         4: return
      LineNumberTable:
        line 5: 0
}
```

从上面Father的反编译代码可以知道类在声明成员变量并赋初值的执行是在构造方法中执行,编译器会在调用父类的构造方法之后立即初始化这些变量.对于类变量编译器会为这些变量的初始化单独创建一个方法来初始化,此外可以发现静态变量FATHER_STATIC_FINAL没有出现在任何一个方法中,这是由于static final这种修饰变量称之为编译时常量,这种数据变量的值在编译时就已确定,所有它不需要任何代码来动态的初始化,这也就是为什么我们使用Father.FATHER_STATIC_FINAL使用该变量时,Father类没有被初始化的原因.**JVM规范对于static final同时修饰变量的说明如下**

>If the ACC_STATIC flag in the access_flags item of the field_info structure is set, then the field represented by the field_info structure is assigned the value represented by its ConstantValue attribute as part of the initialization of the class or interface declaring the field (§5.5). This occurs prior to the invocation of the class or interface initialization method of that class or interface

注意最后一句**该变量的赋值在所在接口、类的初始化之前完成**

## 2 finnaly执行顺序

测试代码如下

```
public class Main {
	public static void main(String[] args) {
		test();
	}
	
	public static int test(){
		try{
			int x=0;
			return x;
		}
		catch(Exception e){
			int y=1;
			return y;
		}
		finally{
			int z=2;
			return 2;
		}
	}
}
```

利用javap反编译的字节码注释如下

```
Classfile /D:/Main.class
  Last modified 2017-5-7; size 508 bytes
  MD5 checksum 8e2f1fb8680f21e07d3357374d813656
  Compiled from "Main.java"
public class main.Main
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   //...
{
//....
  public static int test();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=6, args_size=0
         0: iconst_0  //将常量0送到操作栈顶
         1: istore_0  //将栈顶数据存储到第0个局部变量
         2: iload_0   //将第0个局部变量送到操作栈顶
         3: istore_1  //将常量0存储到第1个局部变量
         4: iconst_2  //将常量2送到操作栈顶
         5: istore_2  //存储栈顶数据到第2个局部变量
         6: iconst_2  //将常量2送到操作栈顶
         7: ireturn   //结束函数调用,推出该栈帧,返回栈顶常量2
         8: astore_0  //将栈顶的exception存到第0个局部变量
         9: iconst_1
        10: istore_1
        11: iload_1
        12: istore_2
        13: iconst_2
        14: istore_3
        15: iconst_2  //将2送到栈顶
        16: ireturn
        17: astore  4 //将栈顶数据存到第4个局部变量中
        19: iconst_2
        20: istore  5
        22: iconst_2  //将2送到栈顶
        23: ireturn
      Exception table:
         from    to  target type
             0     4     8   Class java/lang/Exception //如果0~4行(不包括4)出现异常为Exception异常则跳转到第8行代码
             0     4    17   any    //如果0~4行出现任何异常则跳转到第17行代码
             8    13    17   any
          //此处可见stackoverflowhttp://stackoverflow.com/questions/6386917/strange-exception-table-entry-produced-by-suns-javac 
            17    19    17   any
}
```

从上面的字节码和注释我们会发现finally代码快之所会执行是因为,java编译时将finally代码块中的代码插入到try-catch之间的每一个代码块最后,这样才保证了finally的执行,但是finally代码块是不是一定会执行呢,其实并不然,由于finally代码块是被插入到每个每个代码块最后,所以如果在执行finally代码块代码之前进程退出等情况发生,则finally不会执行,具体见[官网链接finally说明](https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)

## 3 一个关于finally的题目

如下代码的返回值是什么?

```
public static int test(){
	int a = 0;
	try{
		int b = a/0;
	}catch(Exception e){
		a = 30;
		return a;
	}
	finally{
		a = 40;
	}
	return a;
}
```

话不多说,这道题的答案是返回值是30,那么为什么,同样首先看字节码,截取部分如下

```
 public static int test();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=0
         0: iconst_0
         1: istore_0
         2: iload_0
         3: iconst_0
         4: idiv            // a/0 这一步发生异常,根据异常表跳转到12
         5: istore_1
         6: bipush        40
         8: istore_0
         9: goto          29
        12: astore_1        
        13: bipush        30 //将常量30推到栈顶
        15: istore_0         //存储30到第一个临时变量中
        16: iload_0          //将第一个临时变量送到栈顶
        17: istore_2		 //存储栈顶值到第二个临时变量
        18: bipush        40 
        20: istore_0         //将40存储到第一个临时变量
        21: iload_2			 //将第二个临时变量送到栈顶	
        22: ireturn          //返回栈顶值
        23: astore_3
        24: bipush        40
        26: istore_0
        27: aload_3
        28: athrow
        29: iload_0
        30: ireturn
      Exception table:
         from    to  target type
             2     6    12   Class java/lang/Exception
             2     6    23   any
            12    18    23   any
```

显然finally语句虽然会被插入到返回语句之前,但是它不会改变原先的逻辑,而是引入了新的临时变量来处理数据,它们属于不同的作用域.