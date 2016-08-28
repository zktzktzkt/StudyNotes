#ContentProvider文档笔记#
---
##一、文档概述##
API Guides对ContentProvider的描述是这样的

ContentProvider使用来管理对结构化数据的访问，它封装了数据并提供了相应的安全机制，更重要的是它是一个标准的接口，用于当前代码所在的进程访问处于另一个进程的数据。当我们需要与ContentProvider打交道时，我们只需要使用ContentResolver就可以了，这个对象是用来和具体的实现了ContentProvider的类来进行沟通。一般来说如果我们不需要向其他应用分享数据，我们根本不需要使用ContentProvider，但是当我们为自己的应用添加搜索建议或者希望拷贝文件到其他应用时我们可能需要使用它。

同时ContentProvider向应用提供的数据是类似与关系型数据那样的表的形式，它每一列代表一个数据集合的实体，而每一列的每一行数据则是该数据集的一部分。在android.provider包下有一些Android系统自身提供的ContenetProvider接口。

##二、基本的使用方法##

首先先对_ID字段进行说明：这个字段是数据库自动维护的主键，可以不使用这个字段作为主键，但是如果使用ListView来绑定Provider的数据，则必须使用

###1、如何访问ContentProvider的数据###
	
把ContentProvider比作服务器的话，ContentResolver则是访问数据的客户端，该类提供了基本的CRUD方法来操纵数据，仅仅只是需要一个映射到ContentProvider的Uri(**Content URIs**)和简单的SQL信息

###2、什么是Content URIs###

Content URIs是用在Provider标识数据的Uri,Content URIs包含一个标识整个Provider数据集的符号(Authority,系统会根据这个字符串来与系统中存在的Provider的Authority进行对比匹配)和一个指向一个表的名字(Path)以及content://为协议名必须存在(例如:content://user_dictionary/words/4、content://com.android.contacts/contacts)。另外Content URIs还可以增加一个id用于获取一行数据(ContentUris.withAppendedId是一个在URI后面增加Id的工具方法)、Uri和Uri.Builder也有相应的工具类来从字符串中构造一个格式化的Content URIs

###3、从Provider获取数据###
	
首先需要注意的是利用ContentResolver来进行相关数据的操作的方法都是运行在主线程中(一种异步实现的方法是使用CursorLoader)，此外关于访问权限系统不支持运行时申请，必须通过uses-permission在清单文件中申明。

关于数据的检索核心在于Uri和SQL相关的一些知识，一个值得注意的点是利用占位符来避免SQL注入

###4、访问Provider的几种可选形式###
	
a、批量访问:可以借助ContentProviderOperation类(该类包括了几个CRUD方法来生成操作对象)来建立批量访问、操作数据的方法，然后调用ContentResolver的applyBatch来实施

b、异步查询:使用Loader加载数据的方式

c、通过Intent来访问数据：Intent提供了一种间接的访问Provider的方法，甚至允许应用没有访问Provider的权限仍可以访问到数据，要么是通过从一个拥有权限的App中获取一个结果 Intent(这种方式是通过让一个拥有权限的App授予一个临时权限给请求的App,清单文件中需要有相应的声明),要么是通过打开一个拥有权限的App让用户与它交互

###5、ContentProvider中的mime类型###

基本格式是这样的type/subtype,ContentProvider可以返回标准的mime type 或者自定义的mime type ,对于自定义的mime type(vendor-specific)来说，如果是复杂类型的则type值总是vnd.android.cursor.dir(用于多行数据)，否则为vnd.android.cursor.item(用于单行数据),而subtype则是由provider自己定义

##三、创建ContentProvider##
	
###1、创建步骤###

设计数据的存储方式，一般来说是两种形式文件数据、结构化数据->提供具体的实现类->提供标志字符串,包括Content URIs、列名(还可以提供权限等等相关的信息)

###2、设计的数据的存储方式###

如果是使用数据库，务必提供一个主键(使用 BaseColumns._ID是一个不错的选择，作为数据绑定到ListView之类的控件比较有用)

###3、设计标志串###

对于Authority一般使用这种方式com.example.<appname>.provider，对于路径则是一般采用Authority append path的方式来映射到独立的表，此外对于这种比较难以处理的字符串映射路径和id的形式，Android 提供了UriMatcher这个帮助类的实现一个方便的映射关系。该类可以为一个标志串提供一个整型值作为一个标志，例如

```

	//假设authority代表一个Authority字符串(例如：com.chen.app.provider)
	 sURIMatcher.addURI(uthority, "people", 1);//为content://com.chen.app.provider/people路径生成了一个标志id 1
	 //为content://com.chen.app.provider/people/(id)为people表中任意的id生成字符串，#表示任意数字串，即匹配单行内容
     sURIMatcher.addURI(uthority, "people/#", 2);
     //匹配phones/filter路径下的任意uri
     sURIMatcher.addURI("contacts", "phones/filter/*", PHONES_FILTER);
```

使用通配符的含义为：

*：匹配由任意长度的任何有效字符组成的字符串
	
# ：匹配由任意长度的数字字符组成的字符串	

###4、MIME类型的实现###
	
对于标准类型则是直接返回标准的mime类型，而对于用户自定义的数据库的表类型则是返回Android中定义的类型，如下规则

类型：vnd.android.cursor.dir：多行数据，vnd.android.cursor.item：单行数据

子类型：应该为程序特有的部分规则为 vnd.<name>.<type>

官网上的一个例子

如果提供程序的权限是 com.example.app.provider，并且它公开了一个名为 table1 的表，则 table1 中多个行的 MIME 类型是：vnd.android.cursor.dir/vnd.com.example.provider.table1

对于 table1 的单个行，MIME 类型是：vnd.android.cursor.item/vnd.com.example.provider.table1

最后由于getStreamTypes是获取文件的mime类型，所以一般它返回的是标准的mime类型，如果不想支持则直接返回null

###5、provider元素###

该标签提供了ContentProvider的实现类的声明、读写权限、路径权限、临时权限、是否向第三方应用放数据的标志等等关于ContentProvider的一些详细信息。

##三、关于联系人的Provider##

```
	//一个查询联系人的例子
	public void queryContacts(){
        ContentResolver resolver=getContentResolver();
        Cursor cursor=resolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI,null,null,null,null);
        while (cursor!=null&&cursor.moveToNext()){
            for(int i=0;i<cursor.getColumnCount();i++){
                Log.e("tag",cursor.getColumnName(i)+"="+cursor.getString(i));
            }
            cursor.moveToNext();
        }
        cursor.close();
    }

```

##总结##

总之ContentProvider是一个数据提供器，对于该组件运用主要包括ContentProvider类的实现、映射路径的选择(content://协议)、UriMatcher帮助类的使用、mime类型的固定写法、_ID列名的使用、权限的定义。
