# JNI数据类型

JNI数据类型分为：基本数据类型、引用数据类型、Field and Method IDs。基本数据类型即布尔类型、整型、浮点类型、Void类型；引用类型有类、对象、字符串、数组等；Field and Method IDs比较特殊，它们用于在JNI中表示Java代码中的成员和方法的ID，通过JNI函数Get到这些ID后即可以通过另外的JNI函数操作对应的成员(Field)或者方法(Method)。

### JNI基本数据类型 <a href="#jni-ji-ben-shu-ju-lei-xing" id="jni-ji-ben-shu-ju-lei-xing"></a>

| Java Type | Native Type | Description    |
| --------- | ----------- | -------------- |
| boolean   | jboolean    | unsigned 8bits |
| byte      | jbyte       | unsigned 8bits |
| char      | jchar       | signed 16bits  |
| short     | jshort      | signed 16bits  |
| int       | jint        | signed 32bits  |
| long      | jlong       | signed 64bits  |
| float     | jfloat      | 32bits         |
| double    | jdouble     | 64bits         |
| void      | void        | N/A            |

JNI\_TRUE,JNI\_FALSE是JNI中定义的宏，用于表示true/false。

### JNI引用类型 <a href="#jni-yin-yong-lei-xing" id="jni-yin-yong-lei-xing"></a>

| Java Type | Native Type | Description |
| --------- | ----------- | ----------- |
| Object    | jobject     | Java对象      |
| Class     | jclass      | 类           |
| String    | jstring     | 字符串         |
| xxxx      | jxxxxArray  | 数组          |

> Note：Java中的Array（如byte\[], char\[],int\[],long\[],object\[]）类型在JNI中对应jxxxxArray，xxxx是java中数组的类型，如jbyteArray,jcharArray,jintArray,jlongArray,jobjectArray。

JNI引用类型的继承关系如下

### Field and Method IDs <a href="#field-and-method-ids" id="field-and-method-ids"></a>

Field and Method IDs是基本的C指针类型：

```
    typedef struct _jfieldID * jfieldID;
    typedef struct _jmethodID * jmethodID;
```

jfieldID和jmthodID可用于获取类中的成员变量和成员函数的标识，然后通过标识来操作成员变量(读写)和成员函数(调用)。

这里展示一个基础的用法：

```
  jint Java_com_goodix_jni_JNI_Foo(JNIEnv *env, jobject object) {
      jfieldID fid = (*env)->GetFieldID(env, object, "speed", "I")  
      jint  speed = (*env)->GetIntField(env, object, fid);
      speed++;
      (*env)->SetIntField(env, object, fid, speed);
  }

    jint Java_com_goodix_jni_JNI_Foo1(JNIEnv *env, jobject object) {
      jmethodID mid = (*env)->GetMethodID(env, object, "speedUp", "(I)Z");  
      jboolean ok = (*env)->CallBooleanMethod(env, object, mid, 767);
  }
```

### JNI Type Signatures 类型签名 <a href="#jnitypesignatures-lei-xing-qian-ming" id="jnitypesignatures-lei-xing-qian-ming"></a>

JNI类型签名在很多地方需要用到，例如使用RegisterNatives函数注册函数时、使用GetFiledID/GetMethodID时。类型签名是JNI数据类型在JVM中的唯一标识符，使用类型签名可以区分函数形参、返回值，确定变量类型。

| Type Signature     | Java Type   |
| ------------------ | ----------- |
| Z                  | boolean     |
| B                  | byte        |
| C                  | char        |
| S                  | short       |
| I                  | int         |
| J                  | long        |
| F                  | float       |
| D                  | double      |
| Lclass             | 完全限定类       |
| \[type             | 数组类型type\[] |
| (arg-type)ret-type | 函数类型        |

完全限定类如String,Integer的签名为Ljava/lang/String,Ljava/lang/Integer。函数的签名规则为圆括号内按形参顺序依次列出形参的签名列表，返回值签名紧跟圆括号后面。
