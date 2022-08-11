# 定义和注册Native方法

本章介绍定义和注册Native方法的相关知识。在开发Native代码之前，先创建一个Java Class，在这个Class中声明与Native代码中对应的方法：

```
package com.examp.jni;
class HelloJNI {
     static {
         System.loadLibrary("hello-jni");
     }

     public native boolean hello();
}
```

### 定义Native方法 <a href="#ding-yi-native-fang-fa" id="ding-yi-native-fang-fa"></a>

***

定义Native的方法有两种，区别于是映射Native方法与java方法的方式。如果使用函数命名规则来匹配和映射，则只需定义Native方法，JVM在Load Library时通过命名规则去搜索匹配的Native方法；如果采用运行时动态注册（使用RegisterNatives）的方式则需要先定义Native方法然后在JNI\_OnLoad方法内注册NativeMethods。

因为JNI中Native方法与Java方法的映射可通过函数名规则来匹配，如果使用这种方法，可使用javah工具生成C/C++头文件。

```
    javac HelloJni.java
    javah HelloJni
```

当然，如果你熟悉JNI函数名映射规则，你可以自己手动创建。使用函数名称来映射Native方法与Java方法的规则如下：

```
JNIEXPORT ret-type JNICALL Java_{package-and-classname}_{function-name}(jni args)

JNIEXPORT jboolean JNICALL Java_com_examp_jni_HelloJNI_hello(JNIEnv *env, jobject obj);
```

**JNI方法的形参列表至少包含JNIEnv \*, jobjcet两个参数**:

* JNIEnv \*env: JNI Environment的引用，是用env便可以调用所有JNI提供的函数。
* jobjcet obj: 调用该JNI函数的对象的引用。

方法可以有更多的参数，如果有，紧跟以上两个参数后面即可。

### 注册Native方法 <a href="#zhu-ce-native-fang-fa" id="zhu-ce-native-fang-fa"></a>

***

在JNI Overview章节中我们提到，除了使用函数名称映射Native方法与Java方法，还可以在运行时动态注册。使用动态注册的方式简化了函数名称，并且可以动态的更新映射关系。动态创建映射关系步骤如下：

1.  在Native代码中定义方法

    ```
    jboolean hello() {
     // do something
     return JNI_TRUE;
    }
    ```
2.  创建JNINativeMethod数组

    ```
    static JNINativeMethod jniMethods[] = {
     {"hello", "()Z", (void *)hello},
    };
    ```

    应该按照以上格式将所有需要动态注册的方法填充至JNINativeMethod数组，JNINativeMethod结构有三个成员：

    * const char \*name: Java中声明的native方法。
    * const char \*signature：方法的签名。
    * void \*fnPtr： 函数指针
3. 在JNI\_OnLoad中调用RegisterNatives方法注册Natives方法到JVM，建立映射关系。

```
int JNI_OnLoad(JavaVM *vm, void *reserved)
{
    JNIEnv *env；
    if ((*vm)->GetEnv(vm, (void **)&env, JNI_VERSION_1_4) != JNI_OK) {
        return JNI_ERR;
    }

    jclass cls = (*env)->FindClass(env, "LHelloJNI");
    if (cls == NULL)
        return JNI_ERR;

    int len = sizeof(jniMethods) / sizeof(jnimethods[0]);
    (*env)->RegisterNatives(env, cls, jniMethods, len);

    return JNI_VERSION_1_4;
}
```
