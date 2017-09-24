# Java Annotation Processor

## Annotation基础

注解基础部分主要是对官方文档的一些翻译总结,喜欢看官方文档的同学可以移步[官方教程](https://docs.oracle.com/javase/tutorial/java/annotations/)查看

### 1、what is annotation

>Annotations, a form of metadata, provide data about a program that is not part of the program itself. Annotations have no direct effect on the operation of the code they annotate.

<p>&nbsp;&nbsp;注解-一种元数据,提供了一中不属于程序本身的程序的数据,注解对它批注的代码没有直接的影响.虽然注解本身对于程序没有直接,但是如果我们将注解搭配其他的工具一起使用就能让程序简洁许多</p>

### 2、注解的用处

官方的tutorial给出了三种用法

>Annotations have a number of uses, among them:
Information for the compiler — Annotations can be used by the compiler to detect errors or suppress warnings.
Compile-time and deployment-time processing — Software tools can process annotation information to generate code, XML files, and so forth.
Runtime processing — Some annotations are available to be examined at runtime.

* 为编译器提供信息-编译器可以使用注解来检查错误、取消警告等
* 编译时和部署时处理-可以使用程序来处理注解从而生成代码、xml文件等等
* 运行时处理-运行时动态的检查注解信息来执行不同的逻辑

### 3、自定义注解

自定义注解主要包括一下两个方面的

* 注解的元数据-包括注解的作用域、作用对象等
* 注解的元素声明-主要声明注解的属性.注:注解的属性的数据类型只能为**基本类型、字符串、Class类类型、注解、枚举、一维数组**

自定义注解的格式如下

```
@Target(ElementType.TYPE)//表示注解的作用对象
@Retention(RetentionPolicy.CLASS)//表示注解的作用域
public @interface DefinedAnno {
    int value() default -1;//value属性,为该属性赋值时可以省略value字段
    String name();//定义name属性
	int[] ids() default {1,2};//定义ids属性并提供默认值
}
```

此外对于标记性注解还可以使用@Documented注解让注解被文档化

target注解取值如下

* ElementType.ANNOTATION_TYPE-表明注解可以被用于其他注解
* ElementType.CONSTRUCTOR-表明注解可以用于构造函数
* ElementType.FIELD-表明注解可以用于类的成员变量和属性
* ElementType.LOCAL_VARIABLE--表明注解可以用于局部变量
* ElementType.METHOD--表明注解可以用于方法
* ElementType.PACKAGE-表明注解可以用于包的声明
* ElementType.PARAMETER-表明注解可以用于方法参数
* ElementType.TYPE--表明注解可以用于类、接口、枚举等所有Class类型

Retention注解取值如下

* RetentionPolicy.SOURCE–注解的作用域限制在源码.java文件级别,编译器会直接忽略它.
* RetentionPolicy.CLASS–注解的作用域限制在编译期间.class文件级别,JVM运行时会直接忽略. 
* RetentionPolicy.RUNTIME–作用域能够延伸到JVM运行时

## 自定义注解处理器

### 基本自定义注解处理器开发的介绍
这部分主要介绍关于编译时注解处理器开发的基础部分包括项目结构的划分、[基本demo](https://github.com/stdnull/demo/tree/master/apdemo)的制作的一些要点、Anroid Studio中Debug的配置

#### 项目结构的划分
这个可以参见一些优秀的开源库来学习一下,比如butterknife的项目划分-annotation注解、processor处理器、app工程这三个部分单独做一个library,然后依赖关系如下图

![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2017/annotation_project_structure.png)

#### 基本demo制作

1.创建一个自定义类继承自AbstractProcessor,并实现两个方法getSupportedAnnotationTypes(返回需要处理的注解类型)、getSupportedSourceVersion(返回需要支持的java版本),如下

```
@Override
public Set<String> getSupportedAnnotationTypes() {
    Set<String> supports = new HashSet<>();
    supports.add(HelloWorld.class.getCanonicalName());
    return supports;
}
@Override
public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
}
```
2.配置Service

由于注解处理器是在编译时由javac自动调用,所以需要为注解处理器的lib注册自定义的注解处理器的入口,这里可以直接使用google的AutoService库,生成Service的配置文件如下
META-INF
    
    -services

        -javax.annotation.processing.Processor

            -处理器全路径名(com.example.HelloWorldProcessor)

3.最后当然是实现process方法来处理注解,这里做了一个demo可以[移步这里]()参考,通过process的RoundEnvironment roundEnv参数我们能够获取到需要处理的java源文件的整个代码结构(Element具体类型依赖于注解修饰的类型),java将整个java文件用了一系列Element 类的进行抽象,将java文件按层次进行了结构化的划分,有点类似与浏览器中的dom.这一部分可以参考参考链接中的注解处理器教程的链接,简单的API介绍如下

+ PackageElement-代表了一个java包节点的信息,可以使用processingEnv.getElementUtils()获得JavacElements来获取PackageElement实例
+ TypeElement-代表了一个class节点或者interface节点的信息,其中enum类型属于class节点,annotation属于interface节点
+ VariableElement-代表一个变量、枚举常量、方法参数、局部变量、资源变量、异常参数的信息节点
+ ExecutableElement-代表一个方法、构造函数、静态初始化块的信息节点

#### Debug自定义注解处理器

1.在gradle.properties中添加以下属性

org.gradle.daemon=true
org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005

2.在Edit Configuration中配置默认的Remote Configration,如下图 

![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2017/ap_debug_remote_configuration.png)

3.在控制台中运行 gradlew.bat app:compileDebugJavaWithJavac --debug

4.添加断点,执行compleDebugSource task,如下图

![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2017/ap_debug_compile_source.png)

## 参考链接

+ [注解处理器教程](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)

+ [注解处理器Debug](https://stackoverflow.com/questions/8587096/how-do-you-debug-java-annotation-processors-using-intellij)
