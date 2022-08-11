---
description: 原始官方地址：https://developer.android.com/training/articles/perf-jni
---

# 附录3：Google官方JNI规范

JNI 是指 Java 原生接口。它定义了 Android 从受管理代码（使用 Java 或 Kotlin 编程语言编写）编译的字节码与原生代码（使用 C/C++ 编写）互动的方式。JNI 不依赖于供应商，支持从动态共享库加载代码，虽然有时较为繁琐，但效率尚可。

**注意**：由于 Android 采用与 Java 编程语言类似的方式将 Kotlin 编译为适合 ART 的字节码，因此您可以根据 JNI 架构及其相关成本将本页指南应用于 Kotlin 和 Java 编程语言。如需了解详情，请参阅 Kotlin 和 Android。

如果您尚不熟悉 JNI，请通读 [Java 原生接口规范](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)，以了解 JNI 的工作原理及提供的功能。首次阅读时，接口的某些部分可能不会立即显示，因此接下来的几个部分对您来说可能会非常实用。

如需查看全局 JNI 引用并了解这些引用创建和删除的位置，请使用 Android Studio 3.2 及更高版本的**内存性能剖析器**中的 JNI 堆视图。

### 常规提示 <a href="#general-tips" id="general-tips"></a>

尽量减少 JNI 层的占用空间。您需要从几个方面来考虑实现这一点。您的 JNI 解决方案应该尝试遵循以下准则（按重要程度依次列出，从最重要的开始）：

* **尽可能减少跨 JNI 层编组资源的次数。**跨 JNI 层进行编组的费用十分高昂。尝试设计一种接口，尽可能减少需要编组的数据量以及必须进行数据编组的频率。
* **尽可能避免在使用受管理编程语言编写的代码与使用 C++ 编写的代码之间进行异步通信**。这样可使 JNI 接口更易于维护。通常，您可以采用与编写界面相同的编程语言保持异步更新，以简化异步界面更新。例如，最好使用 Java 编程语言在两个线程之间进行回调（其中一个线程发出阻塞 C++ 调用，然后在阻塞调用完成时通知界面线程），而不是通过 JNI 从使用 Java 代码的界面线程调用 C++ 函数。
* **尽可能减少需要接触 JNI 或被 JNI 接触的线程数**。如果您确实需要使用 Java 和 C++ 这两种语言的线程池，请尽量保持在池所有者之间（而不是各个工作器线程之间）进行 JNI 通信。
* **将接口代码保存在少量易于识别的 C++ 和 Java 源位置，以便将来进行重构。**酌情考虑使用 JNI 自动生成库。

### JavaVM 和 JNIEnv <a href="#javavm-and-jnienv" id="javavm-and-jnienv"></a>

JNI 定义了两个关键数据结构，即“JavaVM”和“JNIEnv”。两者本质上都是指向函数表的二级指针。（在 C++ 版本中，它们是一些类，这些类具有指向函数表的指针，并具有每个通过该函数表间接调用的 JNI 函数的成员函数。）JavaVM 提供“调用接口”函数，您可以利用此类来函数创建和销毁 JavaVM。理论上，每个进程可以有多个 JavaVM，但 Android 只允许有一个。

JNIEnv 提供了大部分 JNI 函数。您的原生函数都会收到 JNIEnv 作为第一个参数。

该 JNIEnv 将用于线程本地存储。因此，**您无法在线程之间共享 JNIEnv**。如果一段代码无法通过其他方法获取自己的 JNIEnv，您应该共享相应 JavaVM，然后使用 `GetEnv` 发现线程的 JNIEnv。（假设该线程包含一个 JNIEnv；请参阅下面的 `AttachCurrentThread`。）

JNIEnv 和 JavaVM 的 C 声明与 C++ 声明不同。`"jni.h"` include 文件会提供不同的类型定义符，具体取决于该文件是包含在 C 还是 C++ 中。因此，我们不建议在这两种语言包含的头文件中添加 NIEnv 参数。（换个说法：如果您的头文件需要 `#ifdef __cplusplus`，且该标头中有任何内容引用 JNIEnv，您可能都必须进行一些额外操作。）

### 线程 <a href="#threads" id="threads"></a>

所有线程都是 Linux 线程，由内核调度。线程通常从受管理代码启动（使用 `Thread.start()`），但也可以在其他位置创建，然后附加到 `JavaVM`。例如，可以使用 `AttachCurrentThread()` 或 `AttachCurrentThreadAsDaemon()` 函数附加通过 `pthread_create()` 或 `std::thread` 启动的线程。在附加之前，线程不包含任何 JNIEnv，也**无法调用 JNI**。

通常，最好使用 `Thread.start()` 创建需要调用 Java 代码的任何线程。这样做可以确保您有足够的堆栈空间、属于正确的 `ThreadGroup` 且与您的 Java 代码使用相同的 `ClassLoader`。而且，设置线程名称以在 Java 中进行调试也比通过原生代码更容易（如果您有 `pthread_t` 或 `thread_t`，请参阅 `pthread_setname_np()`；如果您有 `std::thread` 且需要 `pthread_t`，请参阅 `std::thread::native_handle()`）。

附加原生创建的线程会构建 `java.lang.Thread` 对象并将其添加到“主”`ThreadGroup`，从而使调试程序能够看到它。在已附加的线程上调用 `AttachCurrentThread()` 属于空操作。

Android 不会挂起执行原生代码的线程。如果正在进行垃圾回收，或者调试程序已发出挂起请求，则在线程下次调用 JNI 时，Android 会将其挂起。

通过 JNI 附加的线程**在退出之前必须调用 `DetachCurrentThread()`**。如果直接对此进行编码会很棘手，在 Android 2.0 (Eclair) 及更高版本中，您可以先使用 `pthread_key_create()` 定义将在线程退出之前调用的析构函数，之后再调用 `DetachCurrentThread()`。（将该键与 `pthread_setspecific()` 搭配使用，将 JNIEnv 存储在线程本地存储中；这样一来，该键将作为参数传递到您的析构函数中。）

### jclass、jmethodID 和 jfieldID <a href="#jclass-jmethodid-and-jfieldid" id="jclass-jmethodid-and-jfieldid"></a>

如果要通过原生代码访问对象的字段，请执行以下操作：

* 使用 `FindClass` 获取类的类对象引用
* 使用 `GetFieldID` 获取字段的字段 ID
* 使用适当函数获取字段的内容，例如 `GetIntField`

同样，如需调用方法，首先要获取类对象引用，然后获取方法 ID。方法 ID 通常只是指向内部运行时数据结构的指针。查找方法 ID 可能需要进行多次字符串比较，但一旦获取此类 ID，便可以非常快速地进行实际调用以获取字段或调用方法。

如果性能很重要，我们建议您查找一次这些值并将结果缓存在原生代码中。由于每个进程只能包含一个 JavaVM，因此将这些数据存储在静态本地结构中是一种合理的做法。

在取消加载类之前，类引用、字段 ID 和方法 ID 保证有效。只有在与 ClassLoader 关联的所有类可以进行垃圾回收时，系统才会取消加载类，这种情况很少见，但在 Android 中并非不可能。但请注意，`jclass` 是类引用，必须通过调用 `NewGlobalRef` **来保护**它（请参阅下一部分）。

如果您想在加载类时缓存方法 ID，并在取消加载类后重新加载时自动重新缓存方法 ID，那么初始化方法 ID 的正确做法是，将与以下类似的一段代码添加到相应类中：

#### Kotlin <a href="#kotlin" id="kotlin"></a>

```
    companion object {
        /*
         * We use a static class initializer to allow the native code to cache some
         * field offsets. This native function looks up and caches interesting
         * class/field/method IDs. Throws on failure.
         */
        private external fun nativeInit()

        init {
            nativeInit()
        }
    }
    
```

#### Java <a href="#java" id="java"></a>

```
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

在执行 ID 查找的 C/C++ 代码中创建 `nativeClassInit` 方法。初始化类时，该代码会执行一次。如果要取消加载类之后再重新加载，该代码将再次执行。

### 局部引用和全局引用 <a href="#local-and-global-references" id="local-and-global-references"></a>

传递给原生方法的每个参数，以及 JNI 函数返回的几乎每个对象都属于“局部引用”。这意味着，局部引用在当前线程中的当前原生方法运行期间有效。**在原生方法返回后，即使对象本身继续存在，该引用也无效。**

这适用于 `jobject` 的所有子类，包括 `jclass`、`jstring` 和 `jarray`。（启用扩展的 JNI 检查时，运行时会针对大部分引用误用问题向您发出警告。）

获取非局部引用的唯一方法是通过 `NewGlobalRef` 和 `NewWeakGlobalRef` 函数。

如果您希望长时间保留某个引用，则必须使用“全局”引用。`NewGlobalRef` 函数将局部引用作为参数，然后返回全局引用。在调用 `DeleteGlobalRef` 之前，全局引用保证有效。

这种模式通常在缓存 `FindClass` 返回的 jclass 时使用，例如：

```
jclass localClass = env->FindClass("MyClass");
    jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```

所有 JNI 方法都接受局部引用和全局引用作为参数。对同一对象的引用可能具有不同的值。例如，对同一对象连续调用 `NewGlobalRef` 所返回的值可能有所不同。**如需了解两个引用是否引用同一对象，必须使用 `IsSameObject` 函数。**切勿在原生代码中使用 `==` 比较各个引用。

如果使用此符号，**您就不能假设对象引用在原生代码中是常量或唯一值**。在两次调用同一个方法时，表示某个对象的 32 位值可能有所不同；而在连续调用方法时，两个不同的对象可能具有相同的 32 位值。请勿将 `jobject` 值用作键。

程序员需要“不过度分配”局部引用。实际上，这意味着如果您要创建大量局部引用（也许是在运行对象数组时），应该使用 `DeleteLocalRef` 手动释放它们，而不是让 JNI 为您代劳。该实现仅需要为 16 个局部引用保留槽位，因此如果您需要更多槽位，则应该按需删除，或使用 `EnsureLocalCapacity`/`PushLocalFrame` 保留更多槽位。

请注意，`jfieldID` 和 `jmethodID` 属于不透明类型，不是对象引用，且不应传递给 `NewGlobalRef`。函数返回的 `GetStringUTFChars` 和 `GetByteArrayElements` 等原始数据指针也不属于对象。（这些指针可以在线程之间传递，并且在匹配的 Release 调用完成之前一直有效。）

还有一种不寻常的情况值得单独提及。如果使用 `AttachCurrentThread` 附加原生线程，那么在线程分离之前，您运行的代码绝不会自动释放局部引用。您创建的任何局部引用都必须手动删除。通常，在循环中创建局部引用的任何原生代码可能需要执行某些手动删除操作。

请谨慎使用全局引用。全局引用不可避免，但它们很难调试，并且可能会导致难以诊断的内存（不良）行为。在所有其他条件相同的情况下，全局引用越少，解决方案的效果可能越好。

### UTF-8 和 UTF-16 字符串 <a href="#utf-8-and-utf-16-strings" id="utf-8-and-utf-16-strings"></a>

Java 编程语言使用的是 UTF-16。为方便起见，JNI 还提供了使用[修改后的 UTF-8](http://en.wikipedia.org/wiki/UTF-8#Modified\_UTF-8) 的方法。修改后的编码对 C 代码非常有用，因为它将 \u0000 编码为 0xc0 0x80，而不是 0x00。这样做的好处是，您可以依靠以零终止的 C 样式字符串，非常适合与标准 libc 字符串函数配合使用。但缺点是，您无法将任意 UTF-8 数据传递给 JNI 并期望它能够正常工作。

如果可能，使用 UTF-16 字符串执行操作通常会更快。Android 目前不需要 `GetStringChars` 的副本，而 `GetStringUTFChars` 需要分配和转换为 UTF-8。请注意，**UTF-16 字符串不是以零终止的**，并且允许使用 \u0000，因此您需要保留字符串长度和 jchar 指针。

**不要忘记 `Release` 您 `Get` 的字符串**。字符串函数会返回 `jchar*` 或 `jbyte*`，它们是指向原始数据而非局部引用的 C 样式指针。这些指针在调用 `Release` 之前保证有效，这意味着在原生方法返回时不会释放这些指针。

**传递给 NewStringUTF 的数据必须采用修改后的 UTF-8 格式**。一种常见的错误就是从文件或网络数据流中读取字符数据，并在未过滤的情况下将其传递给 `NewStringUTF`。除非您确定数据是有效的 MUTF-8（或 7 位 ASCII，这是一个兼容子集），否则您需要剔除无效字符或将它们相应转换为修改后的 UTF-8 格式。如果不这样做，UTF-16 转换可能会产生意外的结果。CheckJNI 默认状态下为模拟器启用，它会扫描字符串并且在收到无效输入时会中止虚拟机。

### 原始数组 <a href="#primitive-arrays" id="primitive-arrays"></a>

JNI 提供访问数组对象内容的函数。虽然访问对象数组时只能一次访问一个条目，但可以直接读写原始类型的数组，就像它们是在 C 语言中声明的一样。

为了在不限制虚拟机实现的情况下使接口尽可能高效，`Get<PrimitiveType>ArrayElements` 系列调用允许运行时返回指向实际元素的指针，或者分配一些内存并进行复制。无论采用哪种方式，在发出相应的 `Release` 调用之前，返回的原始指针保证有效（这意味着，如果没有复制数据，数组对象的位置将固定不变，并且无法在压缩堆期间重新调整位置）。**您必须 `Release` 自己 `Get` 的每个数组。**此外，如果 `Get` 调用失败，您必须确保自己的代码稍后不会试图 `Release` NULL 指针。

您可以通过传入 `isCopy` 参数的非 NULL 指针来确定是否复制了数据。但这用处不大。

`Release` 调用采用的 `mode` 参数可为三个值中的一个。运行时执行的操作取决于其返回的指针是指向实际数据还是指向数据副本：

* `0`
  * 实际数据：数组对象未固定。
  * 数据副本：已复制回数据。释放了包含相应副本的缓冲区。
* `JNI_COMMIT`
  * 实际数据：不执行任何操作。
  * 数据副本：已复制回数据。**未释放**包含相应副本的缓冲区。
* `JNI_ABORT`
  * 实际数据：数组对象未固定。**未**中止早期的写入数据。
  * 数据副本：释放了包含相应副本的缓冲区；对该副本所做的任何更改都会丢失。

检查 `isCopy` 标记的其中一个原因是，了解您对数组进行更改后是否需要使用 `JNI_COMMIT` 调用 `Release`。如果您要交替进行更改和执行使用数组内容的代码，则可以跳过空操作提交。检查该标记的另一个原因可能是为了有效处理 `JNI_ABORT`。例如，您可能想要获取一个数组、对其进行适当修改、将片段传递给其他函数，然后舍弃所做的更改。如果您知道 JNI 要为您创建新副本，则无需创建另一个“可修改”副本。如果 JNI 要将原始数据传递给您，那么您需要制作自己的副本。

想当然地认为在 `*isCopy` 为 false 时可以跳过 `Release` 调用是一种常见误区（在示例代码中重复出现过这种情况）。事实并非如此。如果没有分配任何副本缓冲区，则必须固定原始内存，并且不能由垃圾回收器移动。

另请注意，`JNI_COMMIT` 标记**不会**释放数组，您最终需要使用其他标记再次调用 `Release`。

### 区域调用 <a href="#region-calls" id="region-calls"></a>

如果您只是想复制数据，使用替代方法进行 `Get<Type>ArrayElements` 和 `GetStringChars` 等调用可能非常有用。请考虑运行以下代码：

```
    jbyte* data = env->GetByteArrayElements(array, NULL);
        if (data != NULL) {
            memcpy(buffer, data, len);
            env->ReleaseByteArrayElements(array, data, JNI_ABORT);
        }
```

这会抓取数组、将第一个 `len` 字节元素复制到数组之外，然后释放数组。`Get` 调用会固定或复制数组内容，具体取决于实现情况。代码复制数据（可能是第二次），然后调用 `Release`；在这种情况下，`JNI_ABORT` 确保不会有机会进行第三次复制。

您还能够以更简单的方式完成相同操作：

```
    env->GetByteArrayRegion(array, 0, len, buffer);
```

这种做法具有诸多优势：

* 需要一个 JNI 调用而不是两个，从而减少开销。
* 不需要固定或额外复制数据。
* 降低程序员出错风险，因为不存在操作失败后忘记调用 `Release` 的风险。

同样，您可以使用 `Set<Type>ArrayRegion` 调用将数据复制到数组中，使用 `GetStringRegion` 或 `GetStringUTFRegion` 将字符复制到 `String` 之外。

### 异常 <a href="#exceptions" id="exceptions"></a>

**在异常挂起时，不得调用大多数 JNI 函数。**您的代码应该会注意到异常（通过函数的返回值 `ExceptionCheck` 或 `ExceptionOccurred`）并返回，或者清除异常并进行处理。

在异常挂起时，您只能调用以下 JNI 函数：

* `DeleteGlobalRef`
* `DeleteLocalRef`
* `DeleteWeakGlobalRef`
* `ExceptionCheck`
* `ExceptionClear`
* `ExceptionDescribe`
* `ExceptionOccurred`
* `MonitorExit`
* `PopLocalFrame`
* `PushLocalFrame`
* `Release<PrimitiveType>ArrayElements`
* `ReleasePrimitiveArrayCritical`
* `ReleaseStringChars`
* `ReleaseStringCritical`
* `ReleaseStringUTFChars`

许多 JNI 调用都会抛出异常，但通常会提供一种更简单的方法来检查失败。例如，如果 `NewString` 返回非 NULL 值，则无需检查异常。但是，如果要调用方法（使用 `CallObjectMethod` 等函数），则必须始终检查异常，因为如果系统抛出异常，返回值将无效。

请注意，解释代码抛出的异常不会展开原生堆栈帧，并且 Android 尚不支持 C++ 异常。JNI `Throw` 和 `ThrowNew` 指令只是在当前线程中设置了异常指针。从原生代码返回到受管理代码后，这些指令会注意到异常并进行相应处理。

原生代码可以通过调用 `ExceptionCheck` 或 `ExceptionOccurred` 来“捕获”异常，然后使用 `ExceptionClear` 进行清除。像往常一样，在未经处理的情况下舍弃异常可能会出现问题。

因为没有可用于操控 `Throwable` 对象本身的内置函数，所以如果您想要获取异常字符串，则需要找到 `Throwable` 类、查找 `getMessage "()Ljava/lang/String;"` 的方法 ID 并调用该方法；如果结果为非 NULL 值，则使用 `GetStringUTFChars` 获取可以传递给 `printf(3)` 或等效函数的内容。

### 扩展的检查 <a href="#extended-checking" id="extended-checking"></a>

JNI 很少进行错误检查。错误通常会导致崩溃。Android 还提供了一种名为 CheckJNI 的模式，其中 JavaVM 和 JNIEnv 函数表指针已切换为在调用标准实现之前执行一系列扩展的检查的函数表。

额外检查包括：

* 数组：尝试分配大小为负值的数组。
* 错误指针：将错误的 jarray/jclass/jobject/jstring 传递给 JNI 调用，或者将 NULL 指针传递给带有不可设为 null 的参数的 JNI 调用。
* 类名称：将类名称的“java/lang/String”样式之外的所有内容传递给 JNI 调用。
* 关键调用：在“关键”get 及其相应 release 之间调用 JNI。
* 直接字节缓冲区：将错误参数传递给 `NewDirectByteBuffer`。
* 异常：在异常挂起时调用 JNI。
* JNIEnv\*：使用错误线程中的 JNIEnv\*。
* jfieldID：使用 NULL jfieldID，或者使用 jfieldID 将字段设置为错误类型的值（例如，尝试将 StringBuilder 分配给 String 字段），或者使用静态字段的 jfieldID 设置实例字段（反之亦然），或者将一个类中的 jfieldID 与另一个类的实例搭配使用。
* jmethodID：在调用 `Call*Method` JNI 时使用错误类型的 jmethodID：返回类型不正确、静态/非静态不匹配、“this”类型错误（对于非静态调用）或类错误（对于静态调用）。
* 引用：对错误类型的引用使用 `DeleteGlobalRef`/`DeleteLocalRef`。
* Release 模式：将错误的 release 模式传递给 release 调用（除 `0`、`JNI_ABORT` 或 `JNI_COMMIT` 之外的内容）。
* 类型安全：从原生方法返回不兼容的类型（例如，从声明返回 String 的方法返回 StringBuilder）。
* UTF-8：将无效的[修改后的 UTF-8](http://en.wikipedia.org/wiki/UTF-8#Modified\_UTF-8) 字节序列传递给 JNI 调用。

（仍未检查方法和字段的可访问性：访问限制不适用于原生代码。）

您可以通过以下几种方法启用 CheckJNI。

如果您使用的是模拟器，CheckJNI 默认处于启用状态。

如果您使用的是已取得 root 权限的设备，则可以使用以下命令序列重新启动运行时，并启用 CheckJNI：

```
adb shell stop
    adb shell setprop dalvik.vm.checkjni true
    adb shell start
```

在以上任何一种情况下，当运行时启动时，您将在 logcat 输出中看到如下内容：

```
D AndroidRuntime: CheckJNI is ON
```

如果您使用的是常规设备，则可以使用以下命令：

```
adb shell setprop debug.checkjni 1
```

这不会影响已经运行的应用，但从那时起启动的任何应用都将启用 CheckJNI。（将属性更改为任何其他值，或者只是重新启动应用都将再次停用 CheckJNI。）在这种情况下，当应用下次启动时，您将在 logcat 输出中看到如下内容：

```
D Late-enabling CheckJNI
```

您还可以在应用清单中设置 `android:debuggable` 属性，以便为您的应用启用 CheckJNI。请注意，Android 构建工具会自动为某些构建类型执行此操作。

### 原生库 <a href="#native-libraries" id="native-libraries"></a>

您可以使用标准 `System.loadLibrary` 从共享库加载原生代码。

实际上，旧版 Android 的 PackageManager 存在错误，导致原生库的安装和更新不可靠。[ReLinker](https://github.com/KeepSafe/ReLinker) 项目能够解决此问题及其他原生库加载问题。

从静态类初始化程序中调用 `System.loadLibrary`（或 `ReLinker.loadLibrary`）。参数是“未修饰”的库名称，因此如需加载 `libfubar.so`，您需要传入 `"fubar"`。

如果您只有一个类具有原生方法，那么合理的做法是应该将对 `System.loadLibrary` 的调用置于该类的静态初始化程序中。否则，您可能需要从 `Application` 进行该调用，这样您就知道始终会加载该库，而且总是会提前加载。

运行时可以通过两种方式找到您的原生方法。您可以使用 `RegisterNatives` 显示注册原生方法，也可以让运行时使用 `dlsym` 进行动态查找。`RegisterNatives` 的优势在于，您可以预先检查符号是否存在，而且还可以通过只导出 `JNI_OnLoad` 来获得规模更小、速度更快的共享库。让运行时发现函数的优势在于，要编写的代码稍微少一些。

如需使用 `RegisterNatives`，请执行以下操作：

* 提供 `JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)` 函数。
* 在 `JNI_OnLoad` 中，使用 `RegisterNatives` 注册所有原生方法。
* 使用 `-fvisibility=hidden` 进行构建，以便只从您的库中导出您的 `JNI_OnLoad`。这将生成速度更快且规模更小的代码，并避免与加载到应用中的其他库发生潜在冲突（但如果应用在原生代码中崩溃，则创建的堆栈轨迹用处不大）。

静态初始化程序应如下所示：

#### Kotlin <a href="#kotlin" id="kotlin"></a>

```
    companion object {
        init {
            System.loadLibrary("fubar")
        }
    }
    
```

#### Java <a href="#java" id="java"></a>

```
    static {
        System.loadLibrary("fubar");
    }
    
```

如果使用 C++ 编写，`JNI_OnLoad` 函数应如下所示：

```
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

如需改为使用“发现”原生方法，您需要以特定方式为其命名（详情请参阅 [JNI 规范](http://java.sun.com/javase/6/docs/technotes/guides/jni/spec/design.html#wp615)）。这意味着，如果方法签名是错误的，您要等到第一次实际调用该方法时才会知道。

从 `JNI_OnLoad` 进行的任何 `FindClass` 调用都会在用于加载共享库的类加载器的上下文中解析类。从其他上下文调用时，`FindClass` 会使用与 Java 堆栈顶部的方法相关联的类加载器，如果没有（因为调用来自刚刚附加的原生线程），则会使用“系统”类加载器。由于系统类加载器不知道应用的类，因此您将无法在该上下文中使用 `FindClass` 查找您自己的类。这使得 `JNI_OnLoad` 成为查找和缓存类的便捷位置：一旦有了有效的 `jclass`，您就可以从任何附加的线程使用它。

### 64 位注意事项 <a href="#64-bit-considerations" id="64-bit-considerations"></a>

为了支持使用 64 位指针的架构，在 Java 字段中存储指向原生结构的指针时，请使用 `long` 字段而不是 `int`。

### 不支持的功能/向后兼容性 <a href="#unsupported-featuresbackwards-compatibility" id="unsupported-featuresbackwards-compatibility"></a>

支持所有 JNI 1.6 功能，但以下情况除外：

* `DefineClass` 未实现。Android 不使用 Java 字节码或类文件，因此传入二进制类数据不起作用。

为了与旧版 Android 向后兼容，您可能需要注意以下几点：

*   **动态查找原生函数**

    在 Android 2.0 (Eclair) 之前，搜索方法名称时，“$”字符未正确转换为“\_00024”。为了解决此问题，需要使用显式注册或将原生方法移出内部类。
*   **分离线程**

    在 Android 2.0 (Eclair) 之前，不可能使用 `pthread_key_create` 析构函数来避免进行“必须在退出前分离线程”检查。（运行时还会使用 pthread key 析构函数，因此它会看看首先要调用哪个函数。）
*   **弱全局引用**

    在 Android 2.2 (Froyo) 之前，没有实现弱全局引用。旧版本会强烈排斥使用弱全局引用。您可以使用 Android 平台版本常量来测试支持情况。

    在 Android 4.0 (Ice Cream Sandwich) 之前，弱全局引用只能传递给 `NewLocalRef`、`NewGlobalRef` 和 `DeleteWeakGlobalRef`。（该规范强烈建议程序员在处理弱全局之前对其创建硬引用，因此这不应该有任何限制。）

    从 Android 4.0 (Ice Cream Sandwich) 开始，弱全局引用可以像任何其他 JNI 引用一样使用。
*   **局部引用**

    在 Android 4.0 (Ice Cream Sandwich) 之前，局部引用实际上是直接指针。Ice Cream Sandwich 增加了为更好的垃圾回收器提供的必要间接支持，但这意味着无法检测到旧版中存在的大量 JNI 错误。如需了解详情，请参阅 [ICS 中的 JNI 局部引用更改](http://android-developers.blogspot.com/2011/11/jni-local-reference-changes-in-ics.html)。

    在 Android 8.0 之前的 Android 版本中，局部引用的数量上限取决于版本特定的限制。从 Android 8.0 开始，Android 支持无限制的局部引用。
*   **使用 `GetObjectRefType` 确定引用类型**

    在 Android 4.0 (Ice Cream Sandwich) 之前，由于使用了直接指针（参见上文），因此无法正确实现 `GetObjectRefType`。我们改为采用启发法，即按顺序查看弱全局表、参数、局部表和全局表。第一次找到您的直接指针时，该方法会报告您的引用类型正好是要检查的类型。举例来说，这意味着如果您对全局 jclass 调用了 `GetObjectRefType`，该 jclass 正好与作为隐式参数传递到静态原生方法的 jclass 相同，那么您就会获得 `JNILocalRefType` 而不是 `JNIGlobalRefType`。

### 常见问题解答：为什么我会收到 `UnsatisfiedLinkError`？ <a href="#faq-why-do-i-get-unsatisfiedlinkerror" id="faq-why-do-i-get-unsatisfiedlinkerror"></a>

在处理原生代码时，经常可以看到如下所示的失败消息：

```
java.lang.UnsatisfiedLinkError: Library foo not found
```

在某些情况下，正如字面意思所说 - 找不到库。在其他情况下，库确实存在但无法通过 `dlopen(3)` 打开，并且可以在有关异常的详细消息中找到失败详细信息。

您可能遇到“找不到库”异常的常见原因如下：

* 库不存在或应用无法访问。请使用 `adb shell ls -l <path>` 检查库的存在情况和相关权限。
* 库不是使用 NDK 构建的。这可能导致对设备上不存在的函数或库产生依赖性。

其他类的 `UnsatisfiedLinkError` 失败消息如下所示：

```
java.lang.UnsatisfiedLinkError: myfunc
            at Foo.myfunc(Native Method)
            at Foo.main(Foo.java:10)
```

在 logcat 中，您将看到以下内容：

```
W/dalvikvm(  880): No implementation found for native LFoo;.myfunc ()V
```

这意味着，运行时试图查找匹配方法但未成功。造成此问题的一些常见原因如下：

* 库未加载。请检查 logcat 输出，以获取有关库加载的消息。
* 名称或签名不匹配，因此找不到该方法。这通常是由以下原因导致的：
  * 对于延迟方法查找，无法使用 `extern "C"` 和相应可见性 (`JNIEXPORT`) 声明 C++ 函数。请注意，在 Ice Cream Sandwich 之前，JNIEXPORT 宏不正确，因此将新 GCC 与旧 `jni.h` 搭配使用不会发挥作用。您可以使用 `arm-eabi-nm` 查看库中显示的符号；如果这些符号看起来支离破碎（类似于 `_Z15Java_Foo_myfuncP7_JNIEnvP7_jclass` 而不是 `Java_Foo_myfunc`），或者如果符号类型是小写字母“t”而不是大写字母“T”，那么您需要调整声明。
  * 对于显式注册，在输入方法签名时会出现轻微错误。请确保传递给注册调用的内容与日志文件中的签名匹配。请记住，“B”是指 `byte`，“Z”是指 `boolean`。签名中的类名称组件以“L”开头，以“;”结尾，使用“/”来分隔软件包/类名称，使用“$”来分隔内部类名称（例如 `Ljava/util/Map$Entry;`）。

使用 `javah` 自动生成 JNI 标头可能有助于避免一些问题。

### 常见问题解答：为什么 `FindClass` 找不到我的类？ <a href="#faq-why-didnt-findclass-find-my-class" id="faq-why-didnt-findclass-find-my-class"></a>

（以下建议的大部分内容同样适用于无法使用 `GetMethodID` 或 `GetStaticMethodID` 找到方法，或者无法使用 `GetFieldID` 或 `GetStaticFieldID` 找到字段的情况。）

确保类名称字符串的格式正确无误。JNI 类名称以软件包名称开头，并用斜线分隔，例如 `java/lang/String`。如果您要查找某个数组类，则需要以适当数量的英文方括号开头，并且还必须用“L”和“;”将该类括起来，因此 `String` 的一维数组将是 `[Ljava/lang/String;`。如果您要查找内部类，请使用“$”而不是“.”。通常，在 .class 文件上使用 `javap` 是查找类的内部名称的好方法。

如果要启用代码缩减，请确保配置要保留的代码。配置适当的保留规则非常重要，否则代码压缩器可能会移除仅在 JNI 中使用的类、方法或字段。

如果类名称形式正确，则可能是您遇到了类加载器问题。`FindClass` 需要在与您的代码关联的类加载器中启动类搜索。它会检查调用堆栈，如下所示：

```
    Foo.myfunc(Native Method)
        Foo.main(Foo.java:10)
```

最顶层的方法是 `Foo.myfunc`。`FindClass` 会查找与 `Foo` 类关联的 `ClassLoader` 对象并使用它。

采用这种方法通常会完成您想要执行的操作。如果您自行创建线程（可能通过调用 `pthread_create`，然后使用 `AttachCurrentThread` 进行附加），可能会遇到麻烦。现在您的应用中没有堆栈帧。如果从此线程调用 `FindClass`，JavaVM 会在“系统”类加载器（而不是与应用关联的类加载器）中启动，因此尝试查找特定于应用的类将失败。

您可以通过以下几种方法来解决此问题：

* 在 `JNI_OnLoad` 中执行一次 `FindClass` 查找，然后缓存类引用以供日后使用。在执行 `JNI_OnLoad` 过程中发出的任何 `FindClass` 调用都会使用与调用 `System.loadLibrary` 的函数关联的类加载器（这是一条特殊规则，用于更方便地进行库初始化）。如果您的应用代码要加载库，`FindClass` 会使用正确的类加载器。
* 通过声明原生方法来获取 Class 参数，然后传入 `Foo.class`，从而将类的实例传递给需要它的函数。
* 在某个便捷位置缓存对 `ClassLoader` 对象的引用，然后直接发出 `loadClass` 调用。这需要花费一些精力来完成。

您可能会发现自己遇到这种情况：需要通过受管理代码和原生代码访问大型原始数据缓冲区。常见示例包括操纵位图或声音样本。您可以通过以下两种基本方法来解决这一问题。

您可以将数据存储在 `byte[]` 中。这样一来，可非常快速地通过受管理代码访问。但对于原生代码，无法保证您能够在不复制数据的情况下访问数据。在某些实现中，`GetByteArrayElements` 和 `GetPrimitiveArrayCritical` 会返回指向托管堆中原始数据的实际指针，但在其他实现中，它将在原生堆上分配缓冲区并复制数据。

另一种方法是将数据存储在直接字节缓冲区中。此类缓冲区可使用 `java.nio.ByteBuffer.allocateDirect` 或 JNI `NewDirectByteBuffer` 函数创建。与常规字节缓冲区不同，存储不在托管堆上分配，并且始终可通过原生代码直接访问（使用 `GetDirectBufferAddress` 获取地址）。根据直接字节缓冲区访问的实现方式，通过受管理代码访问数据可能会非常缓慢。

选择使用哪种方法取决于以下两个因素：

1. 大部分数据是通过使用 Java 编写的代码还是 C/C++ 编写的代码访问？
2. 如果数据最终传递给系统 API，则其必须采用什么格式？（例如，如果数据最终传递给采用 byte\[] 的函数，则在直接 `ByteBuffer` 中处理数据可能并不明智。）

如果这两种方法不分伯仲，请使用直接字节缓冲区。JNI 中直接内置了对这两种方法的支持，并且在未来的版本中，性能应该会得到提升。
