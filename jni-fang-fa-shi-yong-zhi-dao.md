# JNI方法使用指导

### Get ID <a href="#get-id" id="get-id"></a>

***

很多应用场景下，我们需要读写Java中的成员变量或调用Java中成员函数，实现这一功能的第一步是先或者目标（Java变量或函数）的ID。 JNI Environment提供一组用于获取这些ID的接口。

#### GetFieldID, GetStaticFieldID <a href="#getfieldid-getstaticfieldid" id="getfieldid-getstaticfieldid"></a>

```
jfieldID GetFieldID(JNIEnv *env, jclass clazz, const char *name, const char *sig);
jfieldID GetStaticFieldID(JNIEnv *env, jclass clazz, const char *name, const char *sig);
```

GetFieldID返回非静态Java成员变量的ID，通过name和签名来匹配成员变量。GetStaticFieldID返回静态的Java变量的ID。失败返回NULL。

调用这两个方法均会导致未初始化的类被初始化。

#### GetMethodID, GetStaticMethodID <a href="#getmethodid-getstaticmethodid" id="getmethodid-getstaticmethodid"></a>

```
jmethodID GetMethodID(JNIEnv *env, jclass clazz, const char *name, const char *sig);
jmethodID GetStaticMethodID(JNIEnv *env, jclass clazz, const char *name, const char *sig);
```

GetMethodID返回类的非静态成员方法的ID，GetStaticMethodID返回类的静态方法。失败返回NULL。

调用这两个方法均会导致未初始化的类被初始化。

### Call Static/Non-Static method <a href="#call-staticnonstatic-method" id="call-staticnonstatic-method"></a>

***

Native调用Java类的Static/Non-Static方法需通过以下6个JNI方法实现。

```
NativeType CallStatic<type>Method(JNIEnv *env, jclass clazz, jmethodID methodID, ...);
NativeType CallStatic<type>MethodA(JNIEnv *env, jclass clazz, jmethodID methodID, jvalue *args);
NativeType CallStatic<type>MethodV(JNIEnv *env, jclass clazz, jmethodID methodID, va_list args);

NativeType Call<type>Method(JNIEnv *env, jobject  obj, jmethodID methodID, ...);
NativeType Call<type>MethodA(JNIEnv *env, jobject  obj, jmethodID methodID, jvalue *args);
NativeType Call<type>MethodV(JNIEnv *env, jobject  obj, jmethodID methodID, va_list args);
```

Type类型有：Object、Boolean、Byte、Char、Short、Int、Long、Float、Double、Void。注意type指的是返回值的类型。

Static: Method ID对应的方法必须是clazz类或者其派生类的方法，不能是其父类的方法。 Non-Static: Method ID对应的方法需是obj对象所在类或者其派生类的方法，不能是其父类的方法。

以上三个JNI接口方法均用于调用非静态的Java方法，区别在于传递参数的形式不同。使用CallMethod需将所有参数完整列出；使用CallMethodA需将参数保存在jvalue\*列表里；使用CallMethodV需将参数保存在va\_list结构中。

可以看到Call Static Method和Call Nonstatic Method接口函数的第二个形参类型不同，因为static方法不需要有实际对象也可以调用，所以前者是jcalss后者是jobjcet。

### Get/Set Static/Non-Static Field <a href="#getset-staticnonstatic-field" id="getset-staticnonstatic-field"></a>

***

Native对应Java成员变量的操作可通过以下4组JNI方法实现：

```
NativeType GetStatic<type>Field(JNIEnv *env, jclass clazz, jfieldID fieldID);
NativeType Get<type>Field(JNIEnv *env, jobject obj, jfieldID fieldID);

void SetStatic<type>Field(JNIEnv *env, jclass clazz, jfieldID fieldID, NativeType value);
void Set<type>Field(JNIEnv *env, jobject obj, jfieldID fieldID, NativeType value);
```

Type类型有：Object、Boolean、Byte、Char、Short、Int、Long、Float、Double、Void。

### Get/Release Array Elements <a href="#getrelease-array-elements" id="getrelease-array-elements"></a>

***

在Native层需要与Java层通过数据交互数据的应用场景下，JNI提供如下的方法来获得和释放对Java数组的引用。

```
NativeType *Get<PrimitiveType>ArrayElements(JNIEnv *env, ArrayType array, jboolean *isCopy);
void Release<PrimitiveType>ArrayElements(JNIEnv *env, ArrayType array, NativeType *elems, jint mode);
```

PrimitiveType：Boolean，Byte, Char, Short, Int, Long, Float, Double。

#### Get Array Elements <a href="#get-array-elements" id="get-array-elements"></a>

Get方法用于获取对ArrayType的引用，isCopy指示JVM是否创建一份数组的拷贝，如果isCopy不为NULL，当成功创建拷贝的时候isCopy被置为JNI\_TRUE,否则被置为JNI\_FALSE。如果创建了数组的拷贝，则Native代码对数组内容的修改将会在调用Release方法后才会生效。如果isCopy为NULL，则JVM返回实际数组的引用。

#### Release Array Elements <a href="#release-array-elements" id="release-array-elements"></a>

Release方法通知JVM Native代码不再需要操作elems，elems是通过Get方法获得ArrayType引用的指针，JVM将根据参数决定是否要将elems中的修改写回到Java数组中以及如何释放缓冲区。

Mode参数指示如何释放缓冲区，它有如下三种值，在大多数应用场景下传递0使得对数组的修改生效且释放elems缓冲区。

```
mode          actions
0             copy back the content and free the elems buffer
JNI_COMMIT    copy back the content but do not free the elems buffer
JNI_ABORT     free the buffer without copying back the possible changes
```

> Note:Get方法与Release方法需成对出现，因为每调用一次Get方法JVM将创建一份Local Reference，JVM能否创建的Local Reference的数量是有限的，如果大量调用Get方法而不调用Release方法将会导致溢出，JVM会抛出异常致使应用程序停止。

### Get/Set Array Region <a href="#getset-array-region" id="getset-array-region"></a>

***

如果只需要重Java数组中复制数据到Native数组中，则使用GetArrayRegion，这将省去Release操作。

```
void Get<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array, jsize start, jsize len, NativeType *buf);
void Set<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array, jsize start, jsize len, const NativeType *buf);
```

PrimitiveType：Boolean，Byte, Char, Short, Int, Long, Float, Double。

Get方法：

参数array是待Copy的java数组，通常由Java代码通过形参传递到Native方法；start即Copy的起始索引，NativeType是Copy的目的地址。JNI将Copy array数组中的数据到buf所指向的内存里。

Set方法：

参数array Set操作的目标数组，通常由Java代码通过形参传递到Native方法；start即Copy的起始索引，NativeType是Copy的目的地址。JNI将把buf所指向的内存中的数据Copy到array数组中start位置。

> Note: Get Array Region方法不需要任何Release操作。

### Global Weak and Local Reference <a href="#global-weak-and-local-reference" id="global-weak-and-local-reference"></a>

***

当Native代码需要获得Java对象的引用时，则需要使用以下JNI方法。

```
jobject NewGlobalRef(JNIEnv *env, jobject obj);
void DeleteGlobalRef(JNIEnv *env, jobject globalRef);
jobject NewLocalRef(JNIEnv *env, jobject ref);
void DeleteLocalRef(JNIEnv *env, jobject localRef);
jweak NewWeakGlobalRef(JNIEnv *env, jobject obj);
void DeleteWeakGlobalRef(JNIEnv *env, jweak obj);
```

以上API用于创建Global/Local/Weak引用。New操作返回对象的引用，或者返回NULL代表对象已经被释放掉了。

三种类型的区别：

* Global：全局有效，必须显示的Dlete，否则对象将一直不会被JVM回收。
* Local：Local引用只在调用Native方法期间有效，虽然Local Reference会在Native方法调用结束后被释放掉，但任然推荐显示的释放Local Reference。
* Weak：弱全局引用是特殊的全局引用，弱全局引用不增加的引用计数，允许被引用的对象随时被JVM回收掉。因此弱全局引用在使用的时候应先调用`IsSameObject`与NULL比较，若为NULL则代表对象已经被JVM回收掉了。

> 虽然IsSameObject可以用于判断WakeGlobalRef引用的对象是否已经被回收，但它无法阻止JVM回收对象，因此开发人员不应该依赖此方法来决定WakeGlobalRef是否可用，对象任然可能在任意时刻被JVM回收掉。
>
> 官方推荐的做法是：在使用WakeGlobalRef之前先使用GlobalRef或者LocalRef获得对象的引用，如果成功获得（返回值不为NULL）则将阻止JVM回收该对象。

### String Operations <a href="#string-operations" id="string-operations"></a>

***

Native代码中可以针对String做一些操作，包括：

```
jstring NewString(JNIEnv *env, const jchar *unicodeChars, jsize len);
jsize GetStringLength(JNIEnv *env, jstring string);
const jchar * GetStringChars(JNIEnv *env, jstring string, jboolean *isCopy);
void ReleaseStringChars(JNIEnv *env, jstring string, const jchar *chars);
jstring NewStringUTF(JNIEnv *env, const char *bytes);
jsize GetStringUTFLength(JNIEnv *env, jstring string);
const char * GetStringUTFChars(JNIEnv *env, jstring string, jboolean *isCopy);
void ReleaseStringUTFChars(JNIEnv *env, jstring string, const char *utf);
void GetStringRegion(JNIEnv *env, jstring str, jsize start, jsize len, jchar *buf);
jsize GetArrayLength(JNIEnv *env, jarray array);
```

> Note: Java字符和字符串（String，Char）使用Unicode编码，16bits，而C/C++中字符串使用UTF-8编码，8bits，因此在使用时需注意转换。

参考资料：

[Oracle JNI Guide](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)
