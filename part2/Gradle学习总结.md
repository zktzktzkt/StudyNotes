#Gradle学习总结#
---
##一、Gradle基本代码块##
+ 插件声明

用于引入Android插件，这样才能够使用Android相关的Gradle构建

+ Android代码块

该代码块包含全部的Android特有配置，包括defaultConfig代码块(配置应用的核心属性，覆盖相应的Manifest文件)、buildTypes代码块(定义如何构建和打包不同构建类型的应用)

+ dependencies依赖代码块

##二、Gradle依赖##
+ 仓库的概念:
用于提供jar包等第三方依赖,gradle一共支持三种不同类型的仓库Maven、Ivy、静态文件或文件夹。一般在***repositories***代码块里配置

```
repositories {
    jcenter()
}
```

如果使用的是私有的Maven库,可以指定对应的url,并可以提供相应账号、密码凭证.如果是本地Maven库则可以直接使用相对或者绝对路径作为url

```
repositories {
    maven{
    	url "https://...."
    	credentials{
    		username 'user'
    		password 'password'
		}
    }
}
```

对于直接使用本地文件夹的情况则需要使用flatDir代码块

```
repositories {
    flatDir{
        dirs 'arrs'
    }
}
```

+ 添加依赖

首先依赖的一般由三个部分组成:group、name、version，具体添加方式如下

```
compile 'com.google.code.gson:gson.2.3'//从远程仓库获取，缩写形式   
compile group: 'com.google.code.gson', name:'gson',version：'1.9.0'//完整版
compile files('libs/volley.jar')//添加单个文件
compile fileTree('lib')//一次性添加一个文件夹
compile fileTree(dir: 'libs', include: ['*.jar'])//一次性添加文件夹，并指定需要添加的文件后缀
compile(name:'library', ext:'aar')//添加arr库的依赖，需要使用flatDir先添加本地仓库
```

原生依赖库的添加一般是创建jniLibs文件夹放入so库,如果不生效则需要修改源集

```
android{
	sourceSets.main{
		jniLibs.srcDir 'src/main/libs'//将jniLib映射到libs
	}
}
```

+ 依赖版本的动态化

版本化规则:major.minor.patch
1、做不兼容变化major增加
2、向后兼容的方式添加规则minor
3、修复bug时,patch增加

动态化版本

```
dependencies{
	compile 'com.android.support:support-v4:22.2.+'//获取最新的patch
	compile 'com.android.support:support-v7:22.2.2+'//最新的minor，版本号至少为2
	compile 'com.android.support:support-v7:+'//最新版本
}
```

##三、varient##
***Android Studio不会自动创建源集，可以在创建不同文件夹时选择对应的varient***
+ buildTypes:

官网的定义如下
>构建类型定义Gradle在构建和打包应用时使用的某些属性，通常针对开发生命周期的不同阶段进行配置。例如，调试构建类型支持调试选项，使用调试密钥签署APK；而发布构建类型则可压缩、混淆APK以及使用发布密钥签署APK进行分发。

buildTypes定义方式如下

```
android {
    defaultConfig {...}
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        debug {
            applicationIdSuffix ".debug"//修改id可以使得同一个设备上可以同时安装不同构建类型的版本
        }
        jnidebug {
            initWith debug //使用debug属性初始化jnidebug，我们可以选择覆盖我们需要配置的属性
            applicationIdSuffix ".jnidebug"
            jniDebuggable true
        }
    }
}
```

+ product flavor:

>产品风味代表您可以发布给用户的不同应用版本，例如免费和付费的应用版本。您可以将产品风味自定义为使用不同的代码和资源，同时对所有应用版本共有的部分加以共享和重复利用

```
android {
    ...
    defaultConfig {...}//实质上defaultConfig也是一种productFlaors,所以product flavor中可以使用和defaultConfig相同的属性
    buildTypes {...}
    productFlavors {
        demo {
            applicationId "com.example.myapp.demo"
            versionName "1.0-demo"
        }
        full {
            applicationId "com.example.myapp.full"
            versionName "1.0-full"
        }
    }
}
```

+ varient:buildTypes和product flavor结合的结果

+ 源集
>Android Studio 按逻辑关系将每个模块的源代码和资源分组为源集。模块的 main/ 源集包括其所有构建变体共用的代码和资源.
如果不同源集包含同一文件的不同版本，Gradle 将按以下优先顺序决定使用哪一个文件（左侧源集替换右侧源集的文件和设置）：
构建变体 > 构建类型 > 产品风味 > 主源集 > 内容库依赖项

使用方式如下

```
android {
    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'//manifest路径
            java.srcDirs = ['src']//java文件路径
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }

        androidTest.setRoot('tests')
    }
}
```


##四、自定义构建##

+ BuildConfig和资源
Gradle在构建时会生成一个叫做BuildConfig的类，该类包含按照构建类型设置的一些值,使用方式如下

```
buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        mytype{
            initWith release
            applicationIdSuffix '.my'
            versionNameSuffix '-my'
            buildConfigField "Integer" , "MYTYPE","1"//BuildConfig会在对用的构建类型下生成相应的字段
            buildConfigField "String" , "MYNAME" ,"\"type\""//这里的双引号是必须的
            //resValues "string" , "app_name", "mytype"//另一种添加字段的方式，用在product flavor中
        }
    }
```

+ 项目属性

这里说明如何定义Gradle额外的项目属性

1、ext代码块

2、gradle.properties属性文件

3、通过命令行参数

使用方式如下

```
//在项目的build.gradle中配置ext代码块
buildscript {
....
}

allprojects {
    ....
}

ext{
    sdk = 24
}
.....

//gradle.properties文件
local = chinese

//可以使用如下task进行测试
task printAll <<{
    println rootProject.ext.sdk
    println local
    if(project.hasProperty('cmd')){
        println cmd
    }
}

gradlew printAll -Pparam='hello world'//启动时可以使用命令行-P进行自定义参数
```

+ 签名配置

签名配置一般在signingConfigs代码块中

```
signingConfigs{
    release {
        storeFile file("release.jks")//签名文件直接使用路径
        storePassword "password"
        keyAlias "alias"
        keyPassword "password"
    }
    mytype{
        initWith release//直接使用release的签名配置初始化
    }
}
//在相应的buildType或者product flavor中引用
buildTypes {
    mytype{
    	setSigningConfig signingConfigs.release//引用配置
    }
}
```

+ manifest文件操作

除了defaultConfig代码块中对于manifest中关于applicationId等的一些操作外，还可以使用manifest manifestPlaceholders

```
//首先在manifest中使用$引用一个变量
<meta-data
	android:name="key"
    android:value="${VALUE1}"/>
//然后在buildType中或者product flavor中赋值
manifestPlaceholders = [VALUE1 : "value" , VALUE2 : "baidu"]
```

+ Hook构建过程

最简单的Hook到Android构建过程的办法是操控variants,例如修改输出apk名的task
```
android.applicationVariants.all {
    variant->
        variant.outputs.each{
            output->
                def file = output.outputFile
                output.outputFile = new File(file.parent, file.name.replace(".apk","0--1.apk"))
        }
}
```
##五、Tips##

+ 自定义Task中<<表示task中的代码实在执行阶段执行而不是配置阶段执行
+ 使用minifyEnable 和 shrinkResources 来减少apk大小
+ 使用 org.gradle.parallel = true、org.gradle.daemon = true、org.gradle.jvmargs=-Xms256m -Xmx1024m(修改jvm内存)、org.gradle.configureondemand = true(多模块构建时，忽略非必要模块)
