# Annotation 使用备忘

# 概述

本文记录使用 javapoet 以及 auto-service 进行编译时注解的过程以及注意点. 

<!-- more -->

最近又使用了一次编译时注解,期间产生了不少问题.


# 术语的解释

##  Element 
这个代表被注解的元素.这个类有个很重要的方法,getEnclosingElement:这方法的含义是获取 包裹 element 最外围的元素.比如类的最外围的元素是 package.
```java
    PackageElement pkgElement = (PackageElement) element.getEnclosingElement();
```
其他方法都很简单.

## javapoet 库中一些重要的接口和方法

1. TypeName: 对应了 java 代码中的一个类型元素,常用于声明一个方法参数,还有一些 collection 范型使用.
```java
// 用来定一个 ComponentInfo 类的元素类型
   TypeName mComponentInfoClassName = ClassName.get(ComponentInfo.class);
```

2. ParameterizedTypeName 用来声明一个方法的参数.有个 get 方法,这个方法第一个参数是声明原生的类型,后面一个可变参数,声明第一个参数的参数.
```java
// 声明一个参数的类型是 Map<String,List<ComponentInfo>>
 
ParameterizedTypeName paramListComponent = ParameterizedTypeName.get(ClassName.get(List.class), mComponentInfoClassName);
 ParameterizedTypeName moduleLoaderParameter = ParameterizedTypeName.get(
                ClassName.get(Map.class),
                ClassName.get(String.class),
                paramListComponent
        );
```

3. ParameterSpec

这个类代表方法的参数
```java
// 声明一个方法的参数是:final Map<String,List<ComponentInfo>>  targetMap.
 ParameterSpec injectParameterSpec = ParameterSpec.builder(moduleLoaderParameter, "targetMap", Modifier.FINAL).build();

```

4. MethodSpec.Builder
方法的构造器,用来构造一个方法.
```java
// 用来构造一个私有方法,名字叫 inject ,参数是 Map<String,List<ComponentInfo>>  targetMap,String group,String pkg,String name,int type
MethodSpec.Builder injectElementBuilder = MethodSpec.methodBuilder("inject")
                .addModifiers(Modifier.PRIVATE)
                .addParameter(injectParameterSpec)
                .addParameter(String.class, "group")
                .addParameter(String.class, "pkg")
                .addParameter(String.class, "name")
                .addParameter(int.class, "type");
```

这个构造还可以继续添加语句

```java
injectElementBuilder.addStatement("List<$T> list = targetMap.get(name)", ComponentInfo.class)
		 .beginControlFlow("if( list == null )")
                .addStatement("list = new $T<>()", ArrayList.class)
                .addStatement("targetMap.put(group, list)")
                .endControlFlow()
                .addStatement("ComponentInfo info = new ComponentInfo(type, group, pkg, name)")
                .beginControlFlow("try")
                .addStatement(" info.setClazz(Class.forName(pkg + name))")
                .nextControlFlow("catch(Exception e)")
                .addStatement("e.printStackTrace()")
                .endControlFlow()
                .addStatement("list.add(info)");
```

注意其中的两个 ControlFlow .



5. TypeSpec
用来声明一个类的描述
```java
// 声明一个类,类名是 className,里面有两个方法
TypeSpec typeSpec = TypeSpec.classBuilder("className")
                .addModifiers(Modifier.PUBLIC)
                .addMethod(injectElementMethod)
                .addMethod(injectMethodSpec)
                .build();
```


6. JavaFile

代表一个输出的 java 文件.
```java
// 声明一个包名为 : com.steve.pkg 的java 文件.文件描述用的是一个 TypeSpec 
        JavaFile javaFile = JavaFile.builder("com.steve.pkg", typeSpec).build();

```




# 使用的注意事项
1. 新版的 studio 以及不需要 android-apt
2. 对于注解器的依赖可以通过 annotationProcessor project(':processor') 来对注解器工程进行依赖
3. 对于注解配置选项可以在 gradle 中进行配置

```groovy
defaultConfig {
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode rootProject.ext.android.versionCode
        versionName rootProject.ext.android.versionName

        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [moduleName: project.getName()]
            }
        }

    }
```

4. 对于注解器的注册,@AutoService(Processor.class),不要写错了,写成 process
5. 注解器的配置项需要在注解器中声明 @SupportedOptions("moduleName")
6. 注解器中的 log 不要随便打 error,不然就会停止解析,这点和 android 中的 log 有些差异.







