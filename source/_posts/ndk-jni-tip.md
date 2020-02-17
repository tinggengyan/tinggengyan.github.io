---
title: NDK学习之JNI_Tip
date: 2020-02-13 22:47:58
tags:
categories: [Android]
---
# Override
本篇是对于 Google NDK GUIDES 中 JNI tips 的总结,是关于 JNI **开发过程** 中的一些原则和注意点,没有原理. 所有的内容适用于 Java 和 Kotlin.

## 约定
- managed code (Java/kotlin编写的代码)
- native code  (C/C++编写的代码)

<!-- more -->

# Tips
## General
整体上大的原则是: 尽量减少 JNI 层的操作. 故而有以下3点注意事项,重要性由高到低依次为:

1. JNI 层调用传递的数据尽量少,调用的频率尽量低;
2. JNI Java 调用 native 避免异步调用,异步操作都放在 Java 层.这指的是 JNI 调用,不包含 native 库自身有些异步操作;
3. JNI 操作涉及到的线程越少越好.即使要用线程池,也是由线程池的管理者负责JNI之间的交互,而不是由工作线程直接负责交互;
4. 为了方便维护和重构, 保证JNI相关的代码在固定的位置,容易辨认,且接口尽量少;


## JavaVM & JNIEnv
- 二者本质上都是指向函数表的**指针的指针**.
- 虽然理论上来说,每个进程可以有多个 JavaVM 对象,但是 Android 规定,每个进程只能有一个 JavaVM ;

### 注意点
1. JNIEnv 是个**线程局部变量**,线程不可共享,请勿在线程之间共享 JNIEnv 对象; 如若无其他方式获取 JNIEnv,可以采如下方式;
```C
JNIEnv* env;
vm->AttachCurrentThread(&env, nullptr); // 此处的 vm 即为JavaVM 对象,可以处理成全局单例;
```
2. 由于 JavaVM & JNIEnv 在 C 和 C++ 中的定义是不一样("jni.h" 中包含了二者的不同定义,根据包含"jni.h"的是C还是C++),所以,如果头文件会在 C/C++ 中共享的话,则不能简单的 include,头文件中的方法声明就需要根据C/C++做区分处理;


## Thread
- 所有的线程都是 Linux 线程,都归属内核调度
	* Java/kotlin 创建; 
	* native 创建,然后 *AttachCurrentThread* 到 JavaVM 上; 
- 创建线程最好的方式是通过 Java/kotlin 创建
	* 好处一: 有充足的栈空间;
	* 好处二: 相对 native 创建线程,可以分配正确的 ThreadGroup;
	* 好处三: 通过 JNI 调用的 native 代码可以使用和 Java 中相同的 classloader;
	* 好处四: 相对 native 创建线程,方便设置线程 name,在 debug 的时候很方便;
- native 方式创建线程,并 attach
	* 在 Java 层相应的创建一个 java.lang.Thread 对象;
	* 新建的线程添加进 "main" ThreadGroup,debug 时,就可以看到了;
	* 对一个 AttachCurrentThread 过的线程上再次 AttachCurrentThread 无副作用;
- Android 不会挂起正在执行 native 代码的线程
	* GC 或者 debug 的时候,即使发出了挂起的请求,也只会在下次进行 JNI 请求的时候挂起;
- 已经 attach 过的线程退出时,必须调用 DetachCurrentThread 方法
	* 如果调用不方便,可以通过 pthread_key_create 定义一个 析构函数,在线程退出的时候,调用 DetachCurrentThread;

### 注意点
1. 在 native 层线程在未 attach 之前,是没有 JNIEnv 的,**不能进行 JNI 操作**;
2. 线程资源优先通过 Java 层创建;
3. JNI 调用的 native 方法过于耗时会影响 CPU 调度,间接影响主线程,注意 native 方法的耗时;


## jclass, jmethodID, and jfieldID

- JNI native 层访问 Java 层的**属性**的时候,则需要以下三个步骤;
	* jclass,引用实例对应的 jclass 对象,通过 findclass  获取;
	* jfieldID,属性对应的 ID,通过 GetFieldID  获取;
	* 根据属性的变量类型,通过对应方法获取该对象实例的属性的值,如 GetIntField;

- JNI native 层访问 Java 层的**方法**的时候,则需要以下三个步骤;
	* jclass,引用实例对应的 jclass 对象,通过 findclass  获取;
	* jmethodID,方法对应的 ID,通过 GetMethodID  获取;
	* 根据方法的签名,通过对应方法调用方法,如 CallIntMethod;

- 关于 jfieldID 和 jmethodID 的查找是需要经过字符串比较的,然一旦已经存在 jfieldID 和 jmethodI,获取值/方法调用 是很快的.

- jfieldID 和 jmethodID 本质上,只是指向内部运行时数据结构的指针;

- jfieldID 和 jmethodID  只要 class 没有被卸载,是一直有效的; 但是在 Android 上,虽然概率很低,但是 class 也是可能被卸载的,所以,需要做好安全防护工作;


### 注意点
1. 为了性能考虑,缓存 jfieldID 和 jmethodID; 因为每个进程只有一个 JavaVM,所以在 native 代码中的 static 存储区域中缓存是合适的.
2. 与 jfieldID 和 jmethodID 不同,jclass 是个 class 的引用,缓存的时候,必须用 **GlobalRef** 进行保护;

综上,缓存 ID的最佳方式如下:

```java
  /*
     * We use a class initializer to allow the native code to cache some
     * field offsets. This native function looks up and caches interesting
     * class/field/method IDs. Throws on failure.
     */
	private static native void nativeInit();
    static {
        nativeInit();
    }
```

在 C/C++ 层面实现 nativeInit 方法,进行 ID 的查找和缓存,这样只会在 class 加载时候调用一次,卸载重新加载也会得到调用,可以保证安全;

## Local and global references

该特性适用于所有继承了 jobject 类的对象: jclass,jstring,jarray;
![jobject的继承关系](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/images/types4.gif)
如未特殊说明,以下的对象也都是指的 jobject 或者其子类对应的对象;

### Local references

通过 JNI 传递到 native 方法的所有 object 参数以及 native 方法返回的 object 对象,都是 **"local reference"**. 
特点: 在 **当前线程** 的 **当前 native 方法生命周期内** (条件),该 "local reference" 是有效的.不满足这个条件,即使对象依旧存活,依然是无效的. 换句话说:在 return java 之前都是有效的;

#### Local 的限制
native 函数结束之后, local 引用就会失效,但是有时候需要使用大量的 local 引用.典型的像在遍历数组的时候,需要大量创建 local 引用,这时就需要手动释放(DeleteLocalRef),而不应该依赖 JNI 处理.

- 例外:
一个 native 创建的线程,执行过 AttachCurrentThread 操作,在 detach 之前,程序并不会自动删除 local 引用,创建的任何local 都需要自己手动删除.

##### 8.0 之前(和具体版本相关)
只预留了 16 个了 local 引用的 slot(槽位),超过的,要自己手动释放,否则会crash.也可以使用 EnsureLocalCapacity/PushLocalFrame 来增加槽位.
实测下来: 每个槽位对应 32 个引用,所以,16个槽位,可以存放 512 个 local 引用;
##### 8.0 之后
不限制数量.

### global references

global 正好是为了突破 local 所产生的限制: 当前线程 与 当前 native 方法;
通过  NewGlobalRef 和 NewWeakGlobalRef (可以接收 local 和 global 引用作为参数) 来创建 global 引用,只有调用在 DeleteGlobalRef 之后才会失效;

### 引用的适用范围

对于接收引用的 native 方法,可以接收 local 引用 和 global 引用,除了生命周期以外,用法一致;

#### 版本限制
从 Android4.0 开始,weak global 引用才可像其他的引用一样使用,在此之前,只可用于 NewLocalRef, NewGlobalRef, and DeleteWeakGlobalRef.

### 引用之间的比较

对于指向**相同对象**的**不同引用**的**值是很可能不一样**的.例如,针对同一个对象连续调用 NewGlobalRef 返回的引用,值就可能不同.所以对于两个不同的引用,判断是否指向同一个对象,用 **IsSameObject** 函数判断,千万不要用 **==** .

- 特性带来的影响:
1. 不能假设 native 层中的对象引用是常量或者唯一的; 
2. 同一个方法的两次调用,表示对象的引用可能是不同;
3. 不同对象的引用可能具有相同的值;

故而,切勿将 jobject 作为键;

### 注意点: 
1. 引用仅针对 jobject 及其子类. 而 jfieldID 和  jmethodID 不适用,不应该传递给 NewGlobalRef
2. GetStringUTFChars 和 GetByteArrayElements 返回的是原始数据指针,非对象引用,他们可以在线程间传递,在执行对应的 release 之前,一直有效
3. 总的来说, native 代码中创建的 local 引用,及时的显式 delete
4. 谨慎使用全局引用,太多的全局引用会导致调试困难
5. 引用是否指向同一个对象,用 IsSameObject 方法
6. 典型的使用代码:
```C++
jclass localClass = env->FindClass("MyClass");
jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```


## UTF-8 and UTF-16 strings

### Java 与 JNI 编码不一致
- Java 中用的字符编码是 UTF-16
- JNI 为了方便起见,采用的是 Modified UTF-8(将 \u0000 编码成 *0xc0 0x80* ,而不是 *0x00*,这样得到的字符串,就是一个 C-style 的字符串)


#### 利弊
- 优点: JNI 中可以直接用 libc 字符串相关的函数;
- 缺点: 标准的 UTF-8 的数据传递给 JNI 函数时,可能无法正常工作;

## 注意点

1. 如果可行的话,就全部转成 UTF-16,这样操作的最快.
2. GetStringChars:返回的是 UTF-16 的数据,UTF-16 的字符串是没有结尾的符号的,C style 的字符串函数是没法判断结尾的,所以,如果用 UTF-16 的话,需要自己维护一个字符串**长度**和 jchar 指针.
3. GetStringUTFChars:返回的是 Modified UTF-8 的数据,可以直接用 C style 的字符串函数.
4. GetStringChars 返回值是 jchar 指针,GetStringUTFChars 返回的是 char*,都是原始数据的指针,而不是前一个section里的 reference,在调用对应的 release 方法之前,都是有效的,不用担心作用域问题;相应的,不用时,及时 release;
5. NewStringUTF:参数必须是 Modified UTF-8 格式的,切勿将文件流或者网络下载的标准 UTF-8 格式的数据直接传;


## 处理建议
### 策略一:
JNI jstring 通过 Java 层的 String 的 getBytes("UTF-8") 方法来获取标准 UTF-8 格式的字符串;
当 JNI 返回 Java 层数据时,Java 层可以通过 String 对应的构造方法处理;
### 策略二:
在 native 层面进行编码的转换,JNI 不变,依旧使用 Modified UTF-8,通过算法处理编码转换.


## Primitive arrays
JNI 提供的数组操作需要一个一个的操作,有些麻烦.原生数组可以使得数组像被 native 中定义的数组一样,可以被直接操作.

为了高效 Get<PrimitiveType>ArrayElements(array,isCopy) 系列的函数,既可以返回真实数组的指针,也可以分配内存,拷贝到 native;
1. 无论哪种,指针在调用 release 之前,都是有效的.
2. 如果未采用复制方式,返回的真实数组指针,那么,数组的对象将会固定不变,即使是在 GC 进行堆压缩的时候.
3. get 的数组,需要进行 release,并且不能对一个空指针进行 release.

release 方法有个 mode 参数,执行的效果取决于 Get<PrimitiveType>ArrayElements 方法返回的指针是指向的原始数据,还是复制的内存拷贝;
1. 0
	* a. Actual: 数组对象取消固定.
	* b. Copy: 数据重新拷贝回去,原先分配的内存空间**释放**.
2. JNI_COMMIT
	* a. Actual: does nothing.
	* b. Copy: 数据重新拷贝回去,原先分配的内存空间**并不释放**.
3. JNI_ABORT
	* a. Actual: 数组对象取消固定. 之前的写入已经生效.
	* b. Copy: 原先分配的内存空间释放,数据操作丢失.

一个常见的错误是: 如果 isCopy 是 false,则可以省略 release 操作,这个是非常错误的做法,因为不进行 release 的话,则原始数据将会一直固定,得不到回收器的回收.
其次需要注意: JNI_ABORT 并不会释放数组,需要以其他的 mode 再次调用 release 进行释放,这个是很容易犯错的;比如, JNI_ABORT 之后,再调用 0;

### 注意点
1. 根据需求,决定 Get<PrimitiveType>ArrayElements 是否 copy 数组到 native 
2. 无论何种方式获取的数组,都需要 release
3. release(JNI_ABORT) 并不会释放数组,需要再调用 release(0)

## Region calls
如对 Get<PrimitiveType>ArrayElements 和 GetStringChars 的需求都是 **copy=true** 的话,则 Region call 会是个不错的替代方案,提供了更多的灵活性和更好的性能.

### 考虑一个场景: 需要字节数组中的 len 长度的部分
1. 采用 Get<PrimitiveType>ArrayElements
```C++
// 
jbyte* data = env->GetByteArrayElements(array, true);
if (data != NULL) {
	memcpy(buffer, data, len); // extra,copy part
    env->ReleaseByteArrayElements(array, data, JNI_ABORT);
}
```
2. 采用 GetByteArrayRegion 
```C++
    env->GetByteArrayRegion(array, 0, len, buffer);
```

#### 对比
| 方案 | 代码书写 | JNI调用次数 | 固定Java数组 |
| :----:| :---- | :---- |:---- |
| 方式一 | 复杂,需要执行额外的一次复制操作 | 2 |固定 |
| 方式二 | 简洁,出错率低 | 1 |不固定|

有 Get,也同样有对应的 Set 方法,用于将数据复制回数组或者字符串;

### 注意点
1. 当需要对数组或者字符串进行**copy**操作时候,优先用对应的 Region 操作


## Exceptions
### 限制
1. 当发生异常的时候,大多数的 JNI 方法将不能调用,只有固定的几个方法能调用,参见 [仍可以调用的方法](https://developer.android.com/training/articlesperf-jni#exceptions_1)
2. 由代码中断触发的异常,并不会释放 native 的栈信息,Android 目前也不支持 C++ 的 Exception; JNI 通过 Throw 和 ThrowNew 指令,只是在当前的线程中设置了一个异常的指针,等到 native 方法结束,返回 Java 层的时候,这时候才会被处理.
3. JNI 无法持有 Throwable 这个对象,如果需要在 native 层处理异常,需要 findclass Java 层的 Throwable 类,通过相关方法处理.

### 处理方式
1. 少部分可以通过检查返回值,检查比较简单,比如 NewString,判断返回值是否为 null,进行判断.
2. 大部分需要主动检查异常,比如 CallObjectMethod 函数,因为一旦抛出异常,此时的返回值是无效的.

### 涉及到的 JNI 方法
5. ExceptionCheck 与 ExceptionOccurred, 进行异常的检查和捕获. 
6. ExceptionClear 可以清除异常,但是清除异常不是一个好的处理手段.



### 注意点
1. 通过 ExceptionCheck 检测是否有异常,通过 Throw 抛出到 Java 层进行处理.
2. 如果异常是可以忽略的,先 ExceptionClear,再继续执行其他 JNI 操作,否则会 crash.


## Extended checking
JNI 对错误的检查很少,所以 Android 提供了一种称为 **CheckJNI** 的模式,通过修改 *JavaVM* 和 *JNIEnv* 的函数表指针,实现在调用所有的 JNI 函数之前,都会进行一系列的检查.

[具体的检查项](https://developer.android.google.cn/training/articles/perf-jni#extended-checking): 

### 注意点
1. 模拟器: 默认开启

2. rooted device
```shell
adb shell stop
adb shell setprop dalvik.vm.checkjni true
adb shell start
```
开启后会在 logcat 里看到 *D AndroidRuntime: CheckJNI is ON*

3. regular device: 不会影响正在运行的App,而且开启时,所有启动的App都会检查.
```shell
// 设备重启后失效
adb shell setprop debug.checkjni 1
```
开启后会在 logcat 里看到 *D Late-enabling CheckJNI*

4. 针对单个App进行检查
android:debuggable 设置为 true 即可,正常的 debug版本不需要手动配置,Android build-tool 会自动设置;


## Native libraries

### 加载动态库的方式
以下以打包出的动态so文件为: **lib名字.so** 为例.

1. 系统默认方式加载
```java
static {
    System.loadLibrary("名字");
}
```

2. 官方推荐 ReLinker 方式
在旧版本 Android 的 PackageManager 有 bug 在 App 升级时 so 库可能没有成功复制到 /data/data/packageName/lib/ 下,导致 "java.lang.UnsatisfiedLinkError",故而 Google 推荐用  [ReLinker](https://github.com/KeepSafe/ReLinker)
```java
ReLinker.loadLibrary(context, "名字");
```

3. Facebook SoLoader
ReLinker 不能解决 so 依赖问题, [SoLoader](https://github.com/facebook/SoLoader) 可以解决这个问题.
PS: 接入复杂,我还没玩过.可以参考 Facebook 的 RN 和 fresco.

### 确保运行时可以查找 native 方法
#### RegisterNatives 显式的注册

1. 实现  JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)
2. 在 JNI_OnLoad 方法中,使用 RegisterNatives 注册所有的 native方法
3. 加参数 -fvisibility=hidden 可以保证 只有 JNI_OnLoad 被导出,这样的 so 文件更小,更快,且能避免和App中加载的其他so冲突,但是这会带来一个问题,crash的时候,栈信息会更少

```C++
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // Find your class. JNI_OnLoad is called from the correct class loader context for this to work.
    jclass c = env->FindClass("com/example/app/package/MyClass");
    if (c == nullptr) return JNI_ERR;

    // Register your class' native methods.
    static const JNINativeMethod methods[] = {
        {"nativeFoo", "()V", reinterpret_cast<void*>(nativeFoo)},
        {"nativeBar", "(Ljava/lang/String;I)Z", reinterpret_cast<void*>(nativeBar)},
    };
    int rc = env->RegisterNatives(c, methods, sizeof(methods)/sizeof(JNINativeMethod));
    if (rc != JNI_OK) return rc;

    return JNI_VERSION_1_6;
}

```

优点: 
- 前端即可检查方法是否存在.
- 可以仅导出 JNI_OnLoad 方法,使得共享库更小,更快.

#### 使用 dlsym 动态查找
1. Java 类中声明一个 native 标识的方法.
2. 借助 AndroidStudio 自动生成对应的 native 方法,方法名的生成规则为: *Java_点全部换成下划线的packageName_methodName*.
目前 AndroidStudio 自动生成这类代码的能力很强了.
```C++
extern "C" JNIEXPORT jstring JNICALL
Java_me_ele_wp_ndkstudy_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

优点:
- 优点在于编写的代码较少,尤其借助 AndroidStudio,快捷方便.
缺点:
- 即使是一个参数的错误,也只能等到运行时调用的时候,才能发现.

### native 中加载 Java 类
- JNI_OnLoad 方法中的 FindClass
FindClass 函数的调用,用来查找和加载 Java 类所用的 classloader 与加载 so 文件的那个类所用的 classloader 是同一个,也就是说,在哪个类加载 so 文件,就用哪个类的 classloader.

- 其他地方 FindClass 函数的调用
1. 使用的是 Java 栈顶关联的 classloader
2. 如果不存在 Java 栈(native 线程,attach 到 VM 上),则使用 system classloader.

> 所以,在 JNI_OnLoad 中,查找出所有的 jclass,并进行缓存,是最好的选择.一旦成功获取 jclass,可以任何线程中共享 jclass;


### 注意点
1. 优先选择 ReLinker 进行 so 文件加载.
2. 如果只有一个类有 native 的方法,so 文件的加载,则可以选择放在在该类的静态代码块中进行加载;否则,请在 Application 中进行加载,以确保 App 调用native 方法前,so 文件已经得到正确的加载.
3. 方法的注册,看自己的选择. RegisterNatives 优点相对明显些,如果 native 方法数量不多,二者皆可.
4. native 如果用到 jclass,建议在 JNI_OnLoad 方法中进行缓存,避免出错.

## 64-bit considerations
### 注意点
1. 为了支持 64 位的架构,Java 层存储 native 层的指针时,需要用 **long** 类型,而不是 **int**类型.

# QA
## UnsatisfiedLinkError 如何处理?

### Library 名字 not found
```C++
java.lang.UnsatisfiedLinkError: Library 名字 not found
```
1. 如日志所述,确实没找到 so 文件;
2. so 文件存在,App 无权访问;
通过 *adb shell ls -l <path>* 检查 so 文件是否存在,并检查App 是否有访问的权限;
3. so 库不是通过 NDK 打包的,库中有些函数,在设备上找不到.

### No implementation found for functionName
```C++
java.lang.UnsatisfiedLinkError: myfunc
        at Foo.myfunc(Native Method)
        at Foo.main(Foo.java:10)
W/dalvikvm(  880): No implementation found for native LFoo;.myfunc ()V

```
1. so 库未成功加载,可以通过 logcat 检查加载 so 库的日志;
2. 方法的名字或者签名不匹配;
	a. 函数未 *extern "C JNIEXPORT*;
	b. 显式注册时,签名不对.
```shell
javap -s JavaClassName
```
这个命令可以检查Java方法的签名.

## FindClass 失败
1. 检查类名,方法名,签名等字符串是否写错,同时检查是否被混淆;
2. classloader 的问题: findclass 想在 native 代码关联的 classloader 中搜索类.如果此时是自己创建的 native 线程,再 attach 到 javavm 上,则会在系统 classloader 中查找,如果是自定义的类,必然失败;

### 解决方案
1. JNI_OnLoad 中执行一次 FindClass 查找，然后缓存类引用,各个线程则可以放心使用,**优先推荐**.
2. 通过声明 native 方法来获取 Class 参数: 声明一个有 class 参数的 native 方法,Java 层将 class 传入.这个有些麻烦.

## native 层和 Java 层共享原始数据
存在以下几种方式

1. 数据转换成 byte 数组,两边都处理 byte 数组
Java 层处理起来是很快的,但是 native 层是无法保证不进行 copy 操作的. GetByteArrayElements  和 GetPrimitiveArrayCritical 可以返回 Java 堆上的原始数据的指针,然而有时候,是会在 native 的堆上分配一块空间,再将数据 copy 到 native 堆上的这块空间.

2. 直接字节缓存. 
用 java.nio.ByteBuffer.allocateDirect,JNI 中的 NewDirectByteBuffer 函数来创建直接字节缓存,这个不像常规的 Java 字节 buffer 分配,这部分内存不是在 Java 堆上分配,这部分内存空间,可以交由 native 直接访问(通过 GetDirectBufferAddress 方法地址).
这个的弊端是: Java 层对这部分数据分访问可能很慢;

使用哪种方法取决于
1. 大部分的数据访问是否是通过 Java/C++ ?
2. 这部分数据最终是否需要传给系统 API?这部分 API 接收的数据格式是什么?(例如，如果数据最终传递给采用 byte[] 的函数，则采用 ByteBuffer 就不合适)

如果二者差不多,优先使用直接字节缓存.

# 参考文献
* [Android JNI Tip](https://developer.android.google.cn/training/articles/perf-jni#top_of_page)
* [Java Native Interface Specification](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)