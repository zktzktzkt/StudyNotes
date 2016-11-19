#Android.mk学习笔记#
---

##一、概述##
Android.mk文件位于jni文件夹下用于向构建系统提供源码和动态库的信息，它是gun makefile的子集，它可以用来定义工程级的设置，譬如Application.mk中对于构建系统、环境变量等没有定义的设置或者用于指定、覆盖单个module的设置。Android.mk的具体作用在于指导构建系统如何将源码打包成一个module(**静态库、动态库、独立的可执行文件**),你只需提供源码的信息，系统会自动引入头文件、明确的依赖等显式信息。

##二、基础##

###android.mk文件的基本构成###

首先一个android.mk必须由定义LOCAL_PATH开始

```

LOCAL_PATH := $(call my-dir)

```

这行代码指出源代码开发路径的位置，其中宏函数 `my-dir` 由构建系统定义提供值，返回当前的目录的位置即android.mk的父目录

next step:声明`CLEAR_VARS`变量，该变量的值由系统提供，代码如下

```

include $(CLEAR_VARS)

```

`CLEAR_VARS`变量指向一个特殊的gun makefile文件，该文件将指导删除很多`LOCAL_XXX`变量，比如LOCAL_MODULE、LOCAL_SRC_FILES、LOCAL_STATIC_LIBRARIES等等，但是**LOCAL_PATH除外**，因为构建系统解析每个构建控制文件(.mk)时都是使用同一个gun makefile文件且上下文(context)是全局的，即变量是全局变量的，所以当我们描述一个module时必须再次声明该变量

next step:通过`LOCAL_MODULE`变量定义module name,它定义、存储了你希望构建的module的名字(每个模块的名字必须唯一且不能有空格)

```

LOCAL_MODULE := hello-jni

```

构建系统最终加上前缀和后缀成为最终生成的module的名字，比如libhello-jni.so,但是如果module已经有lib前缀开始则不会添加

next step:枚举源文件(c or c++ file),使用空格分隔单个文件，例如

```

LOCAL_SRC_FILES := hello.c jni.c

```

final step:帮助系统绑定everything到一起

```

include $(BUILD_SHARED_LIBRARY)

```

`BUILD_SHARED_LIBRARY`变量指向一个gun makefile 脚本用于收集所有我们定义的`LOCAL_XXX`变量的信息(最近的一次include)，该脚本最终决定构建什么，怎么构建

总结下来就是设定路径->清楚临时文件->指定module name ->指定包含文件->使用脚本构建

##三、Variables and Macros(变量和宏)##

###1、简述###

构建系统提供了很多可用于Andorid.mk的变量，大部分变量的值系统都有预分配，少部分由用户指定，同时我们还可以自己定义变量，但是不能与NDK构建系统的保留字相冲,保留字如下

+ LOCAL_开始的变量
+ PRIVATE_、NDK_、APP开始的变量，构建系统内部使用
+ 小写字母名,如my-dir,同样由构建系统内部使用

如果需要自定义变量，Android推荐使用以MY_开头的名字

###2、NDK定义的变量###

这里讨论的是gun make变量在解析你的android.mk文件之前的定义即预定义的值，在特定情况下NDK会解析你的android.mk文件多次并且每次部分变量可能会使用不同的值，部分定义如下

+ CLEAR_VARS

该变量指向的脚本几乎会清除所有LOCAL_XXX的定义，一般在定义一个新的module时使用include引入该脚本，语法如下

```

include $(CLEAR_VARS)

```

+ BUILD_SHARED_LIBRARY

该变量指向一个构建脚本，它会收集所有你定义的LOCAL_XXX变量的值，并依据你列出的源文件来构建目标动态库，需要注意的时include该脚本之前必须定义脚本名(**LOCAL_MODULE**)以及枚举出源文件(**LOCAL_SRC_FILES**)

```
include $(BUILD_SHARED_LIBRARY)

```

一般以此构建的动态库的扩展名为.so

+ BUILD_STATIC_LIBRARY

该变量用于构建静态库，静态库不会被打包但是可以用于生成动态库

```

include $(BUILD_STATIC_LIBRARY)

```

一般以此构建的静态库的扩展名为.a

+ PREBUILT_SHARED_LIBRARY、PREBUILT_STATIC_LIBRARY

可以用包含已经预先编译好的library,

```

include $(PREBUILT_SHARED_LIBRARY)
include $(PREBUILT_STATIC_LIBRARY)

//example
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := foo-prebuilt  
LOCAL_SRC_FILES := libfoo.so //注意这里为so
include $(PREBUILT_SHARED_LIBRARY)

```

+ TARGET_ARCH

它的值是目标CPU架构的名字，它的赋值由android.mk文件的APP_ABI变量定义，系统在解析android.mk文件之前会预读取这个值

+ TARGET_PLATFORM

用于指定构建系统构建目标平台,例如android5.1的API值为22，使用如下

```

TARGET_PLATFORM := android-22

```

+ TARGET_ARCH_ABI

该变量用于指定构建系统需要构建的目标CPU平台，可以同时指定多个以空格分开，CPU与设置的对用关系如下

	ARMv5TE-armeabi
	ARMv7-armeabi-v7a
	ARMv8、AArch64-arm64-v8a
	i686-x86
	x86-64-x86_64
	mips32(r1)-mips
	mips64(r6)-mips64
	All-all


用法如下

```

TARGET_ARCH_ABI := arm64-v8a

```

+ TARGET_ABI

目标Android平台的level和ABI的值的合并，一般用于在真机上测试系统镜像是有用，例如让arm-64位的手机运行Android-22

```

TARGET_ABI := android-22-arm64-v8a

```

###3、描述模块的变量###

这里将要描述的变量用于向构建系统描述模块，需要遵循以下步骤

+ 使用`CLEAR_VARS` 变量指向的脚本来清除变量的值
+ 为描述module的变量赋值
+ 使用BUILD_XXX变量为构建系统设置合适的脚本

各个变量的详细讨论如下

+ LOCAL_PATH

描述当前文件的路径，需要在Android.mk文件的开始处引用,且该变量只需使用一次且不会被`CLEAR_VARS`清除

```

LOCAL_PATH := $(call my-dir)

```

+ LOCAL_MODULE

见名知意改变量描述即将构建的module name,需要在包含其他构建脚本之前声明它且名字不能有空格，同时构建系统会自动为其添加前缀(lib)或者
扩展名(.so或者.a),例如定义一个名为libfoo.so的库可以如下操作

```

LOCAL_MODULE := "foo"

```

+ LOCAL_MODULE_FILENAME

另一种定义module name的方法，该方法不会生成lib前缀

```

LOCAL_MODULE_FILENAME := "libnewfoo"

```

+ LOCAL_SRC_FILES

用于描述构建module需要的源文件(其他头文件依赖会自动识别),可以是相对路径(LOCAL_PATH)也可是绝对路径，必须使用unix的**\**作为路径分割

+ LOCAL_CPP_EXTENSION

该变量用于描述其他非cpp的扩展名的源文件的扩展名

```

LOCAL_CPP_EXTENSION := .cxx .cpp .cc

```

+ LOCAL_CPP_FEATURES

该变量指出c++文件依赖的特定的c++特性，它启用特定的编译flag，相对于直接使用`LOCAL_CPPFLAGS`变量来直接的声明c++特性，该方法更友好，由于该
变量作用域为单个module，而后者作用于整个工程

```

LOCAL_CPP_FEATURES := rtti features

```

+ LOCAL_C_INCLUDES

指定需要的c、c++文件的相对于NDK root的路径，即添加文件搜索路径

```

LOCAL_C_INCLUDES := sources/foo 或者 LOCAL_C_INCLUDES := $(LOCAL_PATH)//foo

````

需要避免LOCAL_CFLAGS or LOCAL_CPPFLAGS之后定义该变量

+ LOCAL_STATIC_LIBRARIES

保存当前modules依赖的静态库，如果当前库为动态库，则会将依赖库链接到目标二进制文件，如果当前库静态库则该变量指出其它依赖于当前库的
module也依赖于这些库

+ LOCAL_SHARED_LIBRARIES

指出当前库runtime时需要依赖的库

+ LOCAL_WHOLE_STATIC_LIBRARIES

用来标志对应的模块的是一个集体，当存在循环引用时比较引用

+ LOCAL_CFLAGS、LOCAL_CPPFLAGS、LOCAL_LDLIBS、LOCAL_LDFLAGS、LOCAL_LDFLAGS

指定一些编译、链接时一些特殊flag

+ LOCAL_ALLOW_UNDEFINED_SYMBOLS

默认情况下构建时如果遇到未定义符号时会抛出一个error，如果设置该变量为true，构建系统则不会抛出该异常，这样可能会导致runtime error

+ LOCAL_ARM_MODE

默认情况下构建的二进制库是16位字长指令，设置该变量可以强制构建为32位字长指令

```

LOCAL_ARM_MODE := arm

```

另外也可以在LOCAL_SRC_FILES变量的文件名中加.asm后缀来实现对指定文件编译为32位字长

> 也可以在Application.mk中使用APP_OPTIM指定arm二进制用于调试，由于调试器无法良好的处理thumb mode的code

+ LOCAL_ARM_NEON

标志是否使用NEON的指令集，为armeabi-v7a架构的CPU所使用(并非所有的v7都支持,需要在运行时检查),类似的也可以使用如下方式来设置单个文件

```

LOCAL_SRC_FILES = foo.c.neon bar.c zoo.c.arm.neon//使用两个后缀时.arm必须在前面

```

+ LOCAL_DISABLE_NO_EXECUTE、LOCAL_DISABLE_RELRO、LOCAL_DISABLE_FORMAT_STRING_CHECKS

一些默认开启的一些特性


###4、NDK提供的函数宏###

这里解释的是NDK写好的函数宏，可以使用`$(call <function>)`去执行，并返回文本结果

+ my-dir
 
 返回当前最近一次包含的makefile文件的路径，一般是当前makefile的路径

 + all-subdir-makefiles

 返回my-dir返回的路径下的所有的Android.mk

 + this-makefile

 当前makefile的路径

 + parent-makefile

 返回当前makefile的文件包含树的父目录即上一级makefile的包含目录,类似的还有`grand-parent-makefile`

 + import-module

 用于根据module name查找和包含另一个module的Android.mk文件

 ```

 $(call import-module,<name>)

 ```