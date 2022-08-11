# Android JNI Tips

更详细的介绍，请参考:

1. [Java Native Interface Specification](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)
2. [Andriod JNI Tips](https://developer.android.com/intl/zh-cn/training/articles/perf-jni.html)

### 1.JavaVM and JNIEnv <a href="#1javavm-and-jnienv" id="1javavm-and-jnienv"></a>

***

JNI有两种关键的数据结构，JavaVM和JNIEnv，两者均为指向VM方法JNI方法的列表的的指针(C++版本中它们是Class，Class的所有成员均为函数指针)。JavaVM提供创建和销毁VM的调用接口，理论上可以创建多个VM，但Android仅仅支持一个VM。JNIEnv提供所有JNI接口函数，Native函数的第一个参数即为JNIEnv。

JNIEnv仅仅用于线程本地使用，不允许多个线程共享一个JNIEnv。

### 2.Threads <a href="#2threads" id="2threads"></a>

***

所有Linux线程均被Kernel调度，通常线程起始于托管代码（可理解为Java代码），使用Thread.start方法启动。但线程也可以在任何其他地方被创建，线程创建（使用thread\_create）之后可以使用JNI的AttachCurrendThread方法附加到JavaVM中。

> 注意：新创建的线程在Attach到JavaVM之前，是没有JNIEnv的，不能执行任何JNI调用。

Attaching一个线程到JavaVM时，将使一个java.lang.Thread被创建和添加到main ThreadGroup。对一个已经Attach到JavaVM的线程调用AttachCurrendThread是无效的。

Android不会Suspend执行Native Code的线程，如果垃圾回收正在进行或者调试器在请求Suspend，Android将在线程下一次调用JNI函数时执行Suspend。

Attach到JavaVM的线程需要在其退出前显示调用DetachCurrentThread。如果实现起来有难度，在Android2.0以上系统内可使用pthread\_create\_key来声明一个析构函数，当线程退出的时候，析构函数将被调用，在析构函数中调用DetachCurrentThread是非常好的选择。

### 3.jclass，jmethodID，jfieldID <a href="#3jclass-jmethodid-jfieldid" id="3jclass-jmethodid-jfieldid"></a>

***

Find Class、Get methodID fieldID需要花费大量的时间来做字符串比较，推荐的做法是事先缓存这些IDs，以加快调用速度。因为Android限制每个进程只有一个JavaVM，因此在方法内定义static局部变量来缓存这些IDs可有效的提升性能。

### 4.Local and Global References <a href="#4local-and-global-references" id="4local-and-global-references"></a>

***

传递给Native方法的参数以及绝大多数JNI函数的返回值都是局部引用，局部引用只在Native函数调用期间有效，当Native函数调用完毕后，局部引用将失效。如果想持有一个对象的全局引用，应使用NewGlobalRef，NewGlobalWeakRef, NewGlobalRef保证全局引用的全局有效性，直到显式调用了DeleteGlobalRef。一旦对某个对象调用了NewGlobalRef，其引用计数将增加，VM将不会对其执行垃圾回收，因此在必要的时刻，应该显式调用DeleteGlobalRef来释放被引用的对象，让其被VM回收。

JNI中可以有多个引用指向同一个对象，如果需要测试两个引用是否指向的是同一个对象，应该使用IsSameObject方法，而不是“==”。

JNI数据类型中，jfieldID以及jmethodID不是引用类型，不能对这两个类型的数据使用NewGlobalRef；由GetStringUTFChars以及GetByteArrayElements返回的原始数据指针也不是Object类型，返回的指针可以在多个线程中使用，直到调用了对应的Release方法。

有一种特殊的情况你需要注意：如果你将一个Native线程通过AttachCurrentThread方法Attach到VM，则该Native线程中的Local Reference始终不会自动free掉（线程Detach时才会free掉Local Reference），因此这种情况下，你需要手动Delete所以你创建的Local Reference。通常，在一个Loop中创建的任何Local Reference也需要手动Delete，因为VM能够创建的Local Reference的数量是有限的。

### 5.UTF-8 and UTF-16 Strings <a href="#5utf8-and-utf16-strings" id="5utf8-and-utf16-strings"></a>

***

Java中Char类型与String类型使用UTF-16编码，而C代码中的Char以及String（字符数组）使用UTF-8编码。为解决这个问题JNI提供GetStringUTFChars函数用于返回UTF-8编码的String，但调用该方法将导致额外的内存分配和转换操作，执行速率不及GetStringChars。

当通过Get方法获取到String后不要忘记调用Release方法。String Get方法返回的C方式的指针jchar _，jbyte_指向的是原始的数据而并非局部引用，因此在调用Release方法之前这些指针都是有效的，同时也以为这，在Native方法Return后VM不会自动释放指向的资源。

不要向NewStringUTF方法传递非UTF-8编码的数据。一个常见的错误是从文件或者网络读取数据然后直接传递给NewStringUTF。除非你非常清楚数据是ASCII格式的，否则你应当对其做适当的转换，NewStringUTF只接受UTF-8编码的字符串。

### 6.Exceptions <a href="#6exceptions" id="6exceptions"></a>

***

在异常已经发生的时候不能调用JNI方法，你的代码应该通过ExceptionCheck和ExceptionOccured方法判断是否有异常发生，如果异常发生你可以：返回、处理异常清除异常。在异常发生的时候可以调用的JNI方法如下：

* DeleteGlobalRef
* DeleteLocalRef
* leteWeakGlobalRef
* ExceptionCheck
* ExceptionClear
* ExceptionDescribe
* ExceptionOccurred
* MonitorExit
* PopLocalFrame
* PushLocalFrame
* Release\ArrayElements
* ReleasePrimitiveArrayCritical
* ReleaseStringChars
* ReleaseStringCritical
* ReleaseStringUTFChars

很多JNI调用都可能会抛出异常，但也有一些简单的检查方法。例如NewString返回非空则不用检查Exception，但是当调用类似CallObjectMethod的方法是则需要检查异常。

注意，Android的JNI目前不支持C++的异常，JNI的Throw和ThrowNew方法仅仅设置当前线程的异常指针，当Native代码返回到VM中时，托管代码会发现和抛出异常。

Native方法可以通过ExceptionCheck和ExceptionOccured发现异常，并通过ExceptionClear方法清除异常。在清除异常前执行JNI方法将会导致程序Crash。

### 7.Extended Checking <a href="#7extended-checking" id="7extended-checking"></a>

***

VM会对JNI调用做一些额外的运行时检查，包括：

* 数组：分配数组的长度为负值。
* 非法指针：传递非法的jarray/jclass/jobject/jstring指针到JNI函数。或者传递NULL指针给一个必须为非NULL的参数。
* 类名：传递非法的类名，正确的类名的写法应类似"java/lang/String"。
* DirectByteBuffer：传递非法的参数到NewDirectByteBuffer。
* 异常：在JNI Exception发生时，不做任何处理继续调用JNI方法。
* JNIEnv\*s：在线程中错误的使用JNIEnv（每个线程都有不同的JNIEnv，不能公用）。
* jfieldID，jmethodID：使用和传递非法的或者不匹配的ID。
* 引用：Delete引用的时候调用的Delete方法不匹配。
* Release Mode：在调用Release方法的时候使用了错误了mode（0，JNI\_COMMIT,JNI\_ABORT）。
* 类型安全：Native方法返回值不匹配。
* UFT-8：传递了无效的UTF-8字符串到JNI方法。

**启动JNI检查的方法**

Emulator 默认已打开

ROOT过的设备

```
adb shell stop
adb shell setprop dalvik.vm.checkjni true
adb shell start
```

常规设备

```
adb shell setprop debug.checkjni 1
```
