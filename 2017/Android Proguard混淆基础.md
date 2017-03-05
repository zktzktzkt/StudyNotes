#Android Proguard混淆基础

##1.Proguard的作用
ProGuard是一种***java .class***处理器，包括以下下四个功能

+ 压缩：移除没有用到的类、方法、变量等
+ 优化：将非入口方法、类设为private、static、final,移除无用的参数、方法内联等
+ 混淆：将类、方法、变量改为无意义的名字
+ 预校验：向class文件中添加校验信息，这主要用于Java ME或者Java 6以上的,对于Android来说是可选的

在AS中打包完成之后，proguard输出的文件包括以下几个(位于/build/outputs/mapping/release/)

+ dump.txt：说明 APK 中所有类文件的内部结构。
+ mapping.txt：提供原始与混淆过的类、方法和字段名称之间的转换。
+ seeds.txt：列出未进行混淆的类和成员。
+ usage.txt：列出从 APK 移除的代码。

##2.Proguard in Android
这节主要描述当我们在gradle中打开minifyEnable的配置项时，Proguard会混淆哪些东西、哪些东西不会混淆

##3.Proguard 常用指令

###3.1 keep options

+ -keep [,modifier,...] class_specification：保留指定的类以及其成员、方法不需要混淆，作为一种入口类

+ -keepclassmembers [,modifier,...] class_specification：保留指定的类成员不被混淆

+ -keepclasseswithmembers [,modifier,...] class_specification：保留满足指定条件的类以及方法不被混淆

+ 上述指令+names为keep...,allowshrinking的缩写,例如-keepnames class_specification 为-keep,allowshrinking class_specification的缩写

+ -printseeds [filename]：将keep指令匹配的类以及成员的信息输出到指定文件

###3.2 信息补充
####3.2.1 modifier修饰符包括以下几种
+ includedescriptorclasses：该修饰符用于确保被keep的类的方法的参数和返回值不会被混淆，文档中提到这在处理native方法时比较有效例如

```
#它能够保证native方法的参数和返回值不变
-keepclasseswithmembernames, includedescriptorclasses class * { 
    native <methods>; 
} 
```

+ allowshrinking：允许被保留的类在shrink步骤被去除掉，通过在keep系列指令后添加

+ allowoptimization: 允许被保留的类在optimization步骤被优化

+ allowobfuscation：允许被保留的类被混淆，通过一个简单的实验可知这个修饰会让keep命令失效，导致类被混淆，岂不是矛盾！

####3.2.2 class_specification格式

class_specification的声明的通用模板与java风格类似，并增加了一些统配符来增强功能

```
[@annotationtype] [[!]public|final|abstract|@ ...] [!]interface|class|enum classname
    [extends|implements [@annotationtype] classname]
[{
    [@annotationtype] [[!]public|private|protected|static|volatile|transient ...] <fields> |
                                                                      (fieldtype fieldname);
    [@annotationtype] [[!]public|private|protected|static|synchronized|native|abstract|strictfp ...] <methods> |
                                                                                           <init>(argumenttype,...) |
                                                                                           classname(argumenttype,...) |
                                                                                           (returntype methodname(argumenttype,...));
    [@annotationtype] [[!]public|private|protected|static ... ] *;
    ...
}]
```
#####细节如下

+ 基本符号：[]:表示内容可选，| ：表示或的关系 ，():表示括号内的内容为一组，! :表示取反

+ class: 该关键字既可以指类也可以值接口、interface关键字只能指接口

+ 所有的classname必须写出完整的类名，例如java.lang.String，内部类则使用$美元符来指定，此外classname还可以通过通配符来进行模糊匹配，具体匹配规则如下

```
?	匹配任意单个的符号但不包括Java包分割符例如"mypackage.Test?" 匹配"mypackage.Test1"但不匹配"mypackage.Test12".
*	匹配任意长度的字符但不包括Java包分割符
**	匹配任意长度的字符且包含Java包分割符
```

+ extends和implements关键字一般用于限制通配符的作用域

+ @ 用于修饰注解annotationtype，其中annotationtype与classname格式相同

+ 字段和方法的指定

```
<init>	指代构造方法
<fields> 指代成员变量
<methods>	指代的方法
*	指代方法或者变量
```

需要注意的是上述方法来指定函数和成员变量都无需指定返回类型，，只有init构造函数有参数表,此外还可以使用正则式来指定方法和成员变量，通配符如下

```
?	匹配方法名的单个符号
*	匹配方法的任意多个符号
```

+ 方法参数中的通配符

``` 
%	匹配基本类型除了void
?	匹配类名中的单个字符
*	匹配类名的任意部分但不包括java包分割符
**	匹配任意类型包括java包名分割符
***	匹配任意类型
...	匹配任意参数列表
```

值得注意的是 ?, *, and ** 不会匹配基本类型只有***会匹配任意纬度的任意类型,例如 

```
"** get*()" matches "java.lang.Object getObject()", but not "float getFloat()", nor "java.lang.Object[] getObjects()".
```

###3.2 Shrinking options

+ -dontshrink 取消压缩操作

+ -printusage [filename] 将压缩了的代码信息输出到指定类

###3.3 Optimization options

+ -dontoptimize 取消优化

+ -optimizationpasses n 执行优化的次数，默认是1次

+ -allowaccessmodification 允许Proguard改变方法的作用域，例如将private 提升为public

+ -mergeinterfacesaggressively 是否允许Proguard需要时将接口进行合并

###3.4 Obfuscation options
+  -dontobfuscate 取消混淆操作

+ -printmapping [filename] 输出混淆的匹配信息到指定文件

+ -applymapping filename 使用指定的map文件进行代码混淆，存在的使用map文件，不存在的使用的新的名字

+ -keeppackagenames [package_filter] 不混淆指定的包名


###3.5 Preverification options
+  -dontpreverify 关闭预校验，对于Android来说可以减少处理时间

###3.6 General options

+  -verbose 指定Proguard在处理时输出更多信息，比如出现异常时输出整个错误路径

+ -dontnote [class_filter] 声明不输出潜在的错误或者配置缺失

+ -dontwarn [class_filter] 声明不输出未找到引用等重要错误的警告

+ -ignorewarnings 输出错误信息，但是忽略错误信息继续执行

+ -dump [filename] 输出处理之后的文件的类结构


