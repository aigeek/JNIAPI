# JNI简介

### JNI Introduction and Overview <a href="#jni-introduction-and-overview" id="jni-introduction-and-overview"></a>

JNI全称Java Native Interface，是Java中的一种编程接口。它用于Java代码与Native代码(C/C++)之间相互访问操作。所谓存在必有理，虽然Java已经非常强大了，可以满足绝大多数应用要求，但是在某些场景下仅仅使用Java是很难完成工作的，比如：

* Java的标准库无法支持平台相关的特性，如操作硬件设备。
* 已有使用Native Language编写的库，希望通过移植，Java能够便捷的访问这些库。
* 对性能和时间要求非常严格的应用场景下，使用Java难以满足要求。

使用JNI编程接口，你可以使用Native代码完成以下功能：

* 创建、引用、操作Java代码中定义的Object。
* Native调用Java的方法。
* Java调用Native方法。
* Catch和Throw异常。

### JNI Interface Function and Pointer <a href="#jni-interface-function-and-pointer" id="jni-interface-function-and-pointer"></a>

***

Native代码通过JNI接口函数访问Java中的特性，例如创建Java对象，获取Java对象的引用或值。JNI接口函数保存在JNI的接口指针中（JNIEnv Pointer）。JNIEnv指针是一个二级指针，它指向一个JNI函数列表，JNI函数列表中保存了所有JNI提供的功能函数的指针。 有关JNIEnv，你应该记住：

* JNI接口指针（JNIEnv）只在当前线程中有效，不能将一个线程中的JNIEnv接口指针传递给另外一个线程使用，这是因为JVM在实现JNI的时候将线程的本地参数保存在JNIEnv接口指针指向的内存区域。
* JNIEnv作为参数传递给Native方法，JVM为每个线程创建单独的JNIEnv接口指针，即多个Java线程调用Native方法，Native方法收到的JNIEnv接口指针是不同的。

> JNI中不将JNI函数定义为静态接口函数而是使用指针访问的原因是为了避免与Native代码之间的命名空间冲突。

### Loading Native Library <a href="#loading-native-library" id="loading-native-library"></a>

***

Native代码被编译成Library，Java代码中通过System.loadLibrary("library-name")来导入Native Library。

```
class Foo {
    static {
        System.loadLibrary("library-name");
        // or System.load("path-to-native-library");
    }

    public native void f(int a);
}
```

传递给System.loadLibrary的参数是Native Library的名字，不用包含后缀，Java将根据OS的类型自动扩展后缀名，如Linux中扩展成.so,Windows中扩展成.dll。也可以使用System.load方法导出完整路径的Native Library。

### Native Method Names <a href="#native-method-names" id="native-method-names"></a>

***

Native方法与Java native函数是一一对应的，目前有两种方法来建立这种链接关系。

1. 通过方法命名规则来匹配和建立静态链接关系。
2. 通过运行时动态注册的方式建立链接关系。

方法一需要Native方法的命名遵守JNI的规则，如下代码片段展示了一个简单的例子，规则简单描述如下：

```
  Java_com_goodix_jni_JNIS_funcname(Object a)
```

* 必须以Java\_开头
* 后面跟native函数完整的包名路径，将.替换成\_
* 最后是native函数名称，名称需一一对应

也可以使用运行时动态注册的方式建立链接，开发人员可以在JNI\_Onload函数中使用RegisterNatives动态注册Native方法与Java native函数的映射关系。
