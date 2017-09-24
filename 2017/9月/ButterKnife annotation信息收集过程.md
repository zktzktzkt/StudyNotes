# ButterKnife annotation信息收集过程

本文主要简单的总结一下自己阅读Butterknife源码过程中的一些收获,包括Butterknife的整个执行流程、一些值得注意的API等

## Butterknife处理流程

了解annotation processor的同学都应该知道,对于butterknife这种注通过注解生成的代码的库的代码处理逻辑的入口都是在processor的process方法中,如下

```
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);//获取注解信息
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();
      JavaFile javaFile = binding.brewJava(sdk, debuggable);／／生成java文件
      //...
        javaFile.writeTo(filer);
      //...
    }
    return false;
  }
```

butterknife的process方法很短,不看别的地方的代码应该也能猜到process方法的逻辑,通过findAndParseTargets方法收集注解信息,然后根据注解信息生成java文件,显然如果我们自己去实现一个这样的库也是我们要解决的两个问题,如何收集用户注解信息、如何根据信息生成java文件,其中后者可以从butterknife的依赖就能看出来生成java文件的能力是依赖与javapoet库.前者butterknife怎么实现的呢,接着从findAndParseTargets方法来看

```
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
    scanForRClasses(env);//首先是扫描R类,作用是获取所有的需要处理的资源的信息,见下文,
    Set<? extends Element> elementList = null;
    //...略,处理各个注解,这里只保留了BindView注解的处理过程,其他的类似
    elementList = env.getElementsAnnotatedWith(BindView.class);
    for (Element element : elementList) {
      try {
        parseBindView(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
    //根据注释处理继承关系
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries = new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();
      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();
      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          entries.addLast(entry);// Has a superclass binding but we haven't built it yet. Re-enqueue for later.
        }
      }
    }
    return bindingMap;
  }
```

逻辑看上去是这样首先butterknife通过scanForRClasses遍历R文件->再通过env.getElementsAnnotatedWith获取注解信息处理各个annotation->最后处理待绑定资源的类之间的继承关系,很自然的逻辑,但是有一个我不理解的地方是第一步的作用,直观的感觉是明明只需要后两部就ok了啊.接着先看scanForRClasses方法的作用,由于scanForRClasses方法没有返回值,根据方法名应该是遍历R文件来填充ButterKnifeProcessor的某些成员变量,先看scanForRClasses方法的代码

```
private void scanForRClasses(RoundEnvironment env) {
    if (trees == null) return;//JavacTrees
    RClassScanner scanner = new RClassScanner();
    for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
      for (Element element : env.getElementsAnnotatedWith(annotation)) {
        JCTree tree = (JCTree) trees.getTree(element, getMirror(element, annotation));////JavacTrees$JCAnnotation
        if (tree != null) { //tree can be null if the references are compiled types and not source
          scanner.setCurrentPackage(elementUtils.getPackageOf(element));//这里set的package获取的是对应的注解声明所在的类的路径,例com.example.butterknife.library.adapter
          tree.accept(scanner);//遍历语法树
        }
      }
    }
    ／／这里的scanner.getRClasses()顾名思义就是用户声明的资源id位于的R类的路径,比如R、R2
    for (Map.Entry<PackageElement, Set<Symbol.ClassSymbol>> packageNameToRClassSet: scanner.getRClasses().entrySet()) {
      PackageElement respectivePackageName = packageNameToRClassSet.getKey();//获取声明注解所在的源码包
      for (Symbol.ClassSymbol rClass : packageNameToRClassSet.getValue()) {//获取响应的R,R2类路径
        parseRClass(respectivePackageName, rClass, scanner.getReferenced());
      }
    }
  }
```

这里也是分两步首先先遍历各个用户声明的注解的语法树,这里使用了javac的api,查了资料表示可以用于查看整个语法树,核心逻辑在RClassScanner的visitSelect方法中,如下

```
private static class RClassScanner extends TreeScanner {
    private final Map<PackageElement, Set<Symbol.ClassSymbol>> rClasses = new LinkedHashMap<>();// Maps the currently evaluated rPackageName to R Classes
    private PackageElement currentPackage;//scanForRClasses方法中set的声明注解的类
    private Set<String> referenced = new HashSet<>();//保存扫描到的资源全名
    public void visitSelect(JCTree.JCFieldAccess jcFieldAccess) {
        Symbol symbol = jcFieldAccess.sym;
        //略...  
        Set<Symbol.ClassSymbol> rClassSet = rClasses.get(currentPackage);
        if (rClassSet == null) {
          rClassSet = new HashSet<>();
          rClasses.put(currentPackage, rClassSet);
        }
        //保存了所有收集到的annotation的值,也就是各类butterknife需要绑定R资源,如果是R2则会被替换为原始的R资源,例如com.example.butterknife.library.R.bool.test_bool
        referenced.add(getFqName(symbol));
        rClassSet.add(symbol.getEnclosingElement().getEnclosingElement().enclClass());//rClassSet存储每个包中的设计R,R2类路径例如android.R
    }
}
```

接着就是利用变量到R路径,解析每个R、R2类java文件的信息,对于每一个R文件的解析过程如下

```
/*
* respectivePackageName-代表正在处理的注解代码所在的包,源代码所在的包
* rClass-代表R或R2类
* referenced-对应项目中Butterknife所使用的所有R资源的id,注意都是原始的R资源名的字符串,R2已被替换
*/
private void parseRClass(PackageElement respectivePackageName, Symbol.ClassSymbol rClass, Set<String> referenced) {
    TypeElement element = rClass;//略去部分...
    JCTree tree = (JCTree) trees.getTree(element);
    if (tree != null) { // tree can be null if the references are compiled types and not source
      IdScanner idScanner = new IdScanner(symbols, elementUtils.getPackageOf(element), respectivePackageName, referenced);
      tree.accept(idScanner);
    } else {
      parseCompiledR(respectivePackageName, element, referenced);//这里解析的是已编译过的资源,目标同上tree.accept(idScanner);
    }
}
```

同样逻辑落在IdScanner,类似与RClassScanner,只不过目标不同,重写的方法不一样

```
private static class IdScanner extends TreeScanner {
    private final Map<QualifiedId, Id> ids;//ButterKnifeProcessor#symbols,其实也是整个scanRClass的核心之处
    private final PackageElement rPackageName;//代表R资源所在的包
    private final PackageElement respectivePackageName;//代表声明注解所在的包
    private final Set<String> referenced;////整个项目butterknife需要的资源全名
    //...
    //这里从debug上看是得到一个个类,比如这里可能获取到R2的内部类attr内部类
    @Override public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
      for (JCTree tree : jcClassDecl.defs) {
        if (tree instanceof ClassTree) {
          ClassTree classTree = (ClassTree) tree;
          String className = classTree.getSimpleName().toString();//资源id所属的内部类名称,如layout
          if (SUPPORTED_TYPES.contains(className)) {//对于支持的资源类进行筛选
            ClassName rClassName = ClassName.get(rPackageName.getQualifiedName().toString(), "R", className);／／内部类路径如com.example.butterknife.library.R.attr
            VarScanner scanner = new VarScanner(ids, rClassName, respectivePackageName, referenced);
            ((JCTree) classTree).accept(scanner);
          }
        }
      }
    }
  }
```

VarScanner类的代码如下

```
private static class VarScanner extends TreeScanner {
    private final Map<QualifiedId, Id> ids;//同样是对标ButterKnifeProcessor#symbols
    private final ClassName className;//当前变量的资源id的内部类全路径,例如com.example.butterknife.library.R.attr
    private final PackageElement respectivePackageName;//代表声明注解所在的包
    private final Set<String> referenced;//整个项目butterknife需要的资源全名
    //例如
    @Override public void visitVarDef(JCTree.JCVariableDecl jcVariableDecl) {
      if ("int".equals(jcVariableDecl.getType().toString())) {
        String resourceName = jcVariableDecl.getName().toString();//资源名例font
        String fqName = getFqName(jcVariableDecl.sym);//资源全名例com.example.butterknife.library.R.attr.font
        if (referenced.contains(fqName)) {//检查确实是butterknife需要绑定,可以筛除对应的无用类型的资源
          int id = Integer.valueOf(jcVariableDecl.getInitializer().toString());//资源的id值
          //此处为module的包名、id为具体资源对应的id
          QualifiedId qualifiedId = new QualifiedId(respectivePackageName, id);
          ids.put(qualifiedId, new Id(id, className, resourceName));//将获取的信息保存到ids中,也就是ButterKnifeProcessor#symbols中
        }
      }
}
```

ok,到这里scanForRClasses也就是看完了,结合scanForRClasses没有返回值,所以scanForRClasses唯一的工作就是通过遍历语法树收集信息,主要利用了javac的trees、TreeScanner等类提供的API,创建到三个scanner类来处理代码RClassScanner、IdScanner、VarScanner从R类到R类内部类到内部类的字段一层层的获取到了需要的信息,信息的数据结构,也就是ButterKnifeProcessor#symbols保存的信息如下
性能优化？
用作map的key的QualifiedId数据结构如下

```
final class QualifiedId {
  final PackageElement packageName;//指示使用资源的类所在的包例如com.example.butterknife.library.adapter
  final int id;／／aapt生成的资源的id值
}
```

map的key的对于的value的数据结构如下

```
final class Id {
  final int value;//同QualifiedId的id,为资源的id值
  final CodeBlock code;//代表java文件的代码片段,指示资源的全名,格式例com.example.butterknife.library.R.bool.test_bool
  final boolean qualifed;//一般是true,如果为false则生成的binding类使用资源的intger id来进行绑定,而不是使用资源名R.bool.test_bool进行绑定
```

ok,到这里基本对scanForRClasses的作用心中有数,接着再看拿到这些信息之后如何使用,也就是注解处理处理的过程,由于Butterknife对于每个注解的处理都是使用一个函数来操作而每个的处理逻辑又有点相似,而BindView注解最为常用,所以以BindView为例

```
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap, Set<TypeElement> erasedTargetNames) {
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();//代表注解所在的类
    TypeMirror elementType = element.asType();//拿到修饰字段的类型,例如android.widget.TextView
    if (elementType.getKind() == TypeKind.TYPEVAR) {
      TypeVariable typeVariable = (TypeVariable) elementType;
      elementType = typeVariable.getUpperBound();
    }
    //一些校验略...
    int id = element.getAnnotation(BindView.class).value();//资源id值
    //cncloseingElement应该代码着当前注解所在的类
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    QualifiedId qualifiedId = elementToQualifiedId(element, id);
    if (builder != null) {//每个builder代表着一个将要生成的binding类
      //getId(qualifiedId)方法中使用了之前scanForRClasses方法收集在symbol中的信息
      String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));//获取在sanRClass中获取的信息,检查同一个类中是否重复绑定了同一个资源id
      if (existingBindingName != null) {//如果不等于空,说明在一个类中同一个id资源被绑定到多个view
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName, enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);//构建信息的数据结构,后续生成java文件的主要类
    }
    String name = simpleName.toString();//资源名,例如main_activity_layout
    TypeName type = TypeName.get(elementType);//view的类型
    boolean required = isFieldRequired(element);//检查是否有Nullable注解,如果有在binding时找不到资源id会抛出异常
    //添加新的待绑定的字段到将生成的类中,name是View变量名,type是其数据类型
    builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required));
    erasedTargetNames.add(enclosingElement);//将处理过的id保存,这里是为了后面处理存在继承关系的类
  }
```

观察这里可以发现这里利用了symbol来实现了资源重复绑定的问题,同时也是构造绑定字段的信息,结合parseBindView能够获取的信息,通过注解element类无法获取到注解中资源的名字,只能获取资源的int值,而在生成的bind类中必须通过资源名R.id.xxx这种形式来绑定而不能通过资源的int来绑定,所以这里就找到了scanRClass方法的核心作用.

基本的处理过程到这里就是差不多了,接下来主要是处理继承关系和java文件的具体生成,有了注解的信息这里应该都能够做到,但是代码的细节、涉及到的javapoet库的api都比较多,这里就略过了,接着列一下我关注的其他的点

## 琐碎的点

+ 包路径的问题

> 这个问题是在创建BindingSet.Builder类时否发现的,这个类包含了Butterknife创建一个binding所需要的所有信息,当然其中一个重要的信息是创建的binding类的路径也就是包名,BindingSet.Builder的构造函数如下

```
static Builder newBuilder(TypeElement enclosingElement) {//enclosingElement元素是注解element的父元素,代表注解所在的类
    TypeMirror typeMirror = enclosingElement.asType();
    //..略
    String packageName = getPackage(enclosingElement).getQualifiedName().toString();//获取注解在包名
    String className = enclosingElement.getQualifiedName().toString().substring(
        packageName.length() + 1).replace('.', '$');//处理内部类,在执行环境中获取的目标路径为$
    ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");
    //略...
    //targetType是需要bind的原始路径名、bindclassname是将要生成的类包名
    return new Builder(targetType, bindingClassName, isFinal, isView, isActivity, isDialog);
  }
```

观察上面构建binding类的类名的过程,有没有发现一个奇怪的点,为了有一个将 `.` 替换成 `$` 的逻辑,通过debug发现,内部类通过注解获取的类名路径父类和内部类之间不是用 `$` 分割而是使用 `.` 分割,而在javac编译之后在运行时查看却是通过A$B的形式去查找类,所以这里必须存在一个替换过程

+ Drawable 获取API 

> ContextCompat方法里面原来有个getDrawable方法,封装了根据SDK版本调用不同API Level的getDrawble方法的逻辑,可以消除强迫症了.

+ TreeScanner

> 个人觉得是一个创建这种自动化框架的一个很有用的API,配合Trees类感觉可以拿到很多信息,还有env.getElementUtils()、env.getTypeUtils();记下来有空用用看
