---
title: Annotation 使用备忘
date: 2016-11-16 23:16:43
tags: [Annotation]
categories: [Programming,Java]
---
# 概述

本文记录注解 Annotation 的概念和使用. 

<!-- more -->

# Annotation 注解

## Why 需要注解

在代码中常有些重复的代码，这些代码纯手工太耗时。可以通过一定的标记，然后处理即可。

## What 是注解? Annotation 分类

1. 标准 Annotation
   包括 Override, Deprecated, SuppressWarnings，是 java 自带的几个注解，他们由编译器来识别，不会进行编译，不影响代码运行。
2. 元 Annotation
   @Retention, @Target, @Inherited, @Documented，它们是用来定义 Annotation 的 Annotation。也就是当我们要自定义注解时，需要使用它们。
3. 自定义 Annotation
   自定义的 Annotation。

### 自定义的注解也分为三类，通过元Annotation - @Retention 定义：

>* @Retention(RetentionPolicy.SOURCE)

    源码时注解，一般用来作为编译器标记。如 Override, Deprecated, SuppressWarnings。

>* @Retention(RetentionPolicy.RUNTIME)

    运行时注解，在运行时通过反射去识别的注解，这种注解最大的缺点就是反射消耗性能。

>* @Retention(RetentionPolicy.CLASS)

    编译时注解，在编译时被识别并处理的注解，相当于自动生成代码，没有反射，和正常的手写代码无二。

## Annotation 的工作原理

APT(Annotation Processing Tool)
根据不同类型的注解，采取不同的处理方式，对于 SOURCE 类型的注解，它只会存在代码中，当进行编译成 class 的时候，就会被抛弃了。
RUNTIME 类型的则一直存到 class 文件中，一直存在虚拟机的运行期。CLASS 类型的注解只存到编译期，会根据 处理器的要求进行处理，生成代码或者其他处理方式，处理完只会，就不会存在了，而如果生成了文件，则会一直存在，被打包。


### 术语解释

- Element:
表示一个程序元素，比如包、类或者方法。每个元素都表示一个静态的语言级构造（不表示虚拟机的运行时构造）。 元素应该使用 equals(Object)方法进行比较。不保证总是使用相同的对象表示某个特定的元素。要实现基于 Element 对象类的操作，可以使用 visitor 或者使用 getKind() 方法的结果。使用 instanceof 确定此建模层次结构中某一对象的有效类未必可靠，因为一个实现可以选择让单个对象实现多个 Element 子接口。

在 JDK 1.6 新增的 javax.lang.model 包中定义了16类 Element，包括了 Java 代码中最常用的元素，如：“包（PACKAGE）、枚举（ENUM）、类（CLASS）、注解（ANNOTATION_TYPE）、接口（INTERFACE）、枚举值（ENUM_CONSTANT）、字段（FIELD）、参数（PARAMETER）、本地变量（LOCAL_VARIABLE）、异常（EXCEPTION_PARAMETER）、方法（METHOD）、构造函数（CONSTRUCTOR）、静态语句块（STATIC_INIT，即static{}块）、实例语句块（INSTANCE_INIT，即{}块）、参数化类型（TYPE_PARAMETER，既泛型尖括号内的类型）和未定义的其他语法树节点（OTHER）”。


- TypeElement ：
TypeElement 表示一个类或接口程序元素。提供对有关类型及其成员的信息的访问。注意，枚举类型是一种类，而注释类型是一种接口.
TypeElement 代表了一个 class 或者 interface 的 element 。 DeclaredType 表示一个类或接口类型，后者(DeclaredType)将成为前者(TypeElement)的一种使用（或调用）。这种区别对于一般的类型是最明显的，对于这些类型，单个元素可以定义一系列完整的类型。 例如，元素 java.util.Set 对应于参数化类型 java.util.Set<String> 和 java.util.Set<Number>（以及其他许多类型），还对应于原始类型 java.util.Set。


#### TypeElement,DeclaredType

- 为什么需要 [TypeElement](http://docs.oracle.com/javase/7/docs/api/javax/lang/model/element/TypeElement.html) 呢？
TypeElement 表示一个类或接口程序元素,重点它是一个 element ，是类或者接口的 element。
我们知道 element 有多达16种类型，这些 element 形式各异。各有各的特点，TypeElement 表示的是类或者接口，对于类或者接口，他有很多独有的信息，比如全路径名，超类等等，而对于 method 就没有这个，方法也有个特有的 element ExecutableElement，出现的原因就同理可得了。

- 为什么需要 [DeclaredType](http://docs.oracle.com/javase/7/docs/api/javax/lang/model/type/DeclaredType.html) 呢？
DeclaredType 表示一个类或接口类型，重点它是一个具体的类型。
DeclaredType 有个方法，asElement()方法，这个方法返回的是一个 element，通过这个方法我们就可以获取一些作为 element 才能获取的信息。比如 MirroredTypeException  异常会携带回 TypeMirror，可以强制转换成 DeclaredType，就可以获取一些信息。


根据目前已有的数据和自身的理解。

```java
TypeMirror typeMirror = element.asType();

DeclaredType declaredType = (DeclaredType) typeMirror;

messager.printMessage(Diagnostic.Kind.NOTE, "Annotation class : typeMirror instanceof DeclaredType = " + (typeMirror instanceof DeclaredType));
// 这里的输出为 true

```
这也就解释了为什么 oracle 文档上说的前者调用后者的意思。


## How 使用，自定义注解

前提：自定义注解一定要是 Java library，不能用 Android library。
IDE： AS 新建一个 Android Project
插件：此外我们还需要另外一个库，这个库是为了在 Android 上只用注解而使用的: android-apt。 这个插件可以自动的帮你为生成的代码创建目录, 让生成的代码编译到APK 里面去, 而且它还可以让最终编译出来的APK里面不包含注解处理器本身的代码。

- 允许配置只在编译时作为注解处理器的依赖，而不添加到最后的 APK 或 library
- 设置源路径，使注解处理器生成的代码能被 Android Studio 正确的引用


### 编译时注解

对于编译时注解，在编译项目之前执行的代码，可以生成代码或者生成其他文件，生成的文件将会被打包进项目，但是之前的注解将会被删除，不会进入 class 文件。

#### 传统方式

1.  新建一个 Java module，设置该 module 的 gradle 文件
```groovy
apply plugin: 'java'
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
}
sourceCompatibility = "1.7"
targetCompatibility = "1.7"
```
同时设置 project 的 gradle
```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}
```

2.  新建一个注解 NameGenerate

```java
@Retention(RetentionPolicy.CLASS)
public @interface NameGenerate {
}
```

3. 接下来就是重点了，需要编写注解的处理器

```java

public class NameGenerateProcessor extends AbstractProcessor {

    public static final String CLASSNAME = "NameGeneateList";
    public static final String PACKAGENAME = "com.lvmama.router";

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        Messager messager = processingEnv.getMessager();

        try {
        	// 以新建文件的文件名为参数新建一个 JavaFileObject 对象
            JavaFileObject f = processingEnv.getFiler().createSourceFile(CLASSNAME);
            Writer w = f.openWriter();
            PrintWriter pw = new PrintWriter(w);
			// 将新建文件的内容就以拼接的方式输入即可。
            pw.println("package " + PACKAGENAME + ";");
            pw.println("\npublic class " + CLASSNAME + " { ");

            for (Element element : env.getElementsAnnotatedWith(NameGenerate.class)) {
                PackageElement packageElement = (PackageElement) element.getEnclosingElement();
                String packageName = packageElement.getQualifiedName().toString();
                TypeElement classElement = (TypeElement) element;
                String className = classElement.getSimpleName().toString();
                String fullClassName = classElement.getQualifiedName().toString();
                pw.print("public String " + className + "=\"" + fullClassName + "\";");
            }

            pw.println("}");
            pw.flush();
            pw.close();

        } catch (IOException x) {
            processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, x.toString());
        }
        return true;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(NameGenerate.class.getCanonicalName());
        return types;
    }
}
```
![传统的结构第一步](https://raw.githubusercontent.com/tinggengyan/tinggengyan.github.io/source/imgur/annotation_tradition_step1.png)
![传统的结构第二步](https://raw.githubusercontent.com/tinggengyan/tinggengyan.github.io/source/imgur/annotation_tradition_step2.png)

### 关键方法 process 解释

```java
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    // annotations 当前注解器需要处理的注解的集合
    // roundEnv 可以用来访问当前的 Round 中的语法树的节点，每个语法树中的节点都表示为一个 element。
}
```


- “processingEnv”
它是 AbstractProcessor 中的一个 protected 变量，在注解处理器初始化的时候（init（）方法执行的时候）创建，继承了 AbstractProcessor 的注解处理器代码可以直接访问到它。它代表了注解处理器框架提供的一个上下文环境，要创建新的代码、向编译器输出信息、获取其他工具类等都需要用到这个实例变量。

- @SupportedAnnotationTypes 和 @SupportedSourceVersion

前者代表了这个注解处理器对哪些注解感兴趣，可以使用星号“*”作为通配符代表对所有的注解都感兴趣，后者指出这个注解处理器可以处理哪些版本的 Java 代码。

- 方法返回值的含义
每一个注解处理器在运行的时候都是单例的，如果不需要改变或生成语法树的内容，process（）方法就可以返回一个值为 false 的布尔值，通知编译器这个 Round 中的代码未发生变化，无须构造新的 JavaCompiler 实例。



### 对于注解循环引起的错误的解释
不做任何处理，直接运行上面的程序的话，就会在控制台输出一个错误。
* Attempt to recreate a file for type com.steve.RouterList *
此处解释引用自《深入Java虚拟机_JVM高级特性与最佳实践》P307。
将插入式注解处理器看做一个插件，如果这些插件在处理注解期间对语法树进行了修改，编译器将回到解析及填充符号表的过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止，每一次的循环称为一个 round。


4. 注册处理器
在 src/main 目录下，新建一个和 Java  文件夹平级的文件夹 "resources", 在 resources 文件夹下新建 META-INF 文件夹，在 META-INF 文件夹下新建 services 文件夹， 在 services 文件夹下新建 javax.annotation.processing.Processor 文件，文件夹的内容就是刚刚编写的处理的全路径名，例如 com.steve.NameGenerateProcessor

5. 使用
有两种方式，一是直接在 gradle 中依赖 注解项目。
```groovy
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(':lib_processor')
}
```
另外一种方式就是直接将注解的项目打出 jar 文件，让 app 依赖这个 jar  文件。

build app 项目，就会生成对应的文件。文件的路径:\app\build\generated\source\apt\debug

想在代码中使用生成的 java 文件很简单，像正常自己写的文件一样，直接引用即可。

#### 借助 Google 和 square 的库
传统的方式，过程比较繁琐，借助 Google  的 auto-service 和 square 的 javapoet 可以省一些事。

- Auto
用来注解 Processor 类，生成对应的 META-INF 的配置信息，省去注册处理器这一步，只要在自定义的 Processor 上面加上 @AutoService(Processor.class)
- javapoet
只是一个方便生成代码的一个库，比起简单的字符串拼接，这个看上去更加友好。

总体来说，和传统的方式区别不大，只是修改编写注解的 module 的依赖即可。

1. 修改 Processor 所在 Java module 的 gradle 依赖。

```java
apply plugin: 'java'

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.google.auto.service:auto-service:1.0-rc2'
    compile 'com.squareup:javapoet:1.7.0'
}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"
```
2. 借助 javapoet 生成 Java 代码
有一点需要说清楚，在 AndroidStudio 中用 javapoet 有问题，并不能完全的支持。[这个 jake 在 github 上有解释](https://github.com/square/javapoet/issues/139)。如果想要完整的使用，最好能使用 intellij idea。
所以这里我们切换到 intellij idea 去开发。

![AdnroidStudio上的报错](https://raw.githubusercontent.com/tinggengyan/tinggengyan.github.io/source/imgur/annotation_error.png)

3. 重点还是在注解处理器的处理
需要注意一点的时，即使在注解处理这个看似特别的程序上，依旧是个 Java 程序，也就必然符合面向对象的原则。我将每个注解我需要的信息封装成一个对象，交给一个工具类，由工具类统一完成代码的生成。
```java
public class NameGenerateAnnotatedClass {

    private TypeElement annotatedClassElement;
    private String packageName;
    private String className;

    public NameGenerateAnnotatedClass(TypeElement annotatedClassElement) {
        this.annotatedClassElement = annotatedClassElement;
        className = annotatedClassElement.getSimpleName().toString();
        packageName = annotatedClassElement.getQualifiedName().toString();
    }

    public String getPackageName() {
        return packageName;
    }

    public String getClassName() {
        return className;
    }

}
```
这个类很简单，就单纯的记录了每个被注解元素的类名和包名，因为待会儿我自动生成的代码，我只需要这个两样。

```java

public class ProcessorUtil {

    public static final String CLASSNAME = "RouterList";
    public static final String PACKAGENAME = "com.lvmama.router";

    public ProcessorUtil(Messager messager) {
        this.messager = messager;
    }

    private Messager messager;
    private ArrayList<NameGenerateAnnotatedClass> list = new ArrayList<>();

    private void error(Element e, String msg, Object... args) {
        messager.printMessage(Diagnostic.Kind.ERROR, String.format(msg, args), e);
    }

   public void generateCode() {
        TypeSpec.Builder builder = TypeSpec.classBuilder(CLASSNAME).addModifiers(Modifier.PUBLIC);
        for (NameGenerateAnnotatedClass annotatedClass : list) {
            FieldSpec fieldSpec = FieldSpec.builder(String.class, annotatedClass.getClassName())
                    .addModifiers(Modifier.PUBLIC, Modifier.STATIC, Modifier.FINAL)
                    .initializer("$S", annotatedClass.getPackageName())
                    .build();
            builder.addField(fieldSpec);
        }
        TypeSpec typeSpec = builder.build();
        JavaFile.Builder javaFileBuilder = JavaFile.builder(PACKAGENAME, typeSpec);
        JavaFile javaFile = javaFileBuilder.build();
        try {
//            javaFile.writeTo(System.out); //输出到控制台
            javaFile.writeTo(filer); //输出到默认的目录
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void addNameGenerateAnnotatedClass(NameGenerateAnnotatedClass item) {
        list.add(item);
    }

    public void clear() {
        list.clear();
    }
}
```
这个类是负责生成代码的，在获取到每个被注解的元素的时候，都会添加到这个类的 list 中，在 generateCode() 方法中生成的代码。
这里用的就是 javapoet 来生成的代码。
- TypeSpec 代表的是一个类。
- FieldSpec 代表了一个类中的字段。
- JavaFile 代表了一个 Java 文件。
javaFile.writeTo 就是将我们自身组装的这些元素输出，javaFile.writeTo(System.out); 是输出到控制台。
javaFile.writeTo(filer); 输出到默认的目录下。


和传统方式一样，运行 build 命令，在生成的目录下就可以看到生成的文件。

![采用 auto-service和javapoet的结构图](https://raw.githubusercontent.com/tinggengyan/tinggengyan.github.io/source/imgur/annotation_withautoservice.png)

### 运行时注解

对于运行时注解都是通过反射来实现的。

### 源码注解

