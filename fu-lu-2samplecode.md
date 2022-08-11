# 附录2：SampleCode

```
#include <jni.h>
#include <stdio.h>
#include <errno.h>

//com/ex/Foo.java

/* Native function */
JNIEXPRORT jboolean JNICALL Java_com_ex_Foo_fooFunc(JNIEnv *env, jobject obj, jint k)
{
    if (k > 0)
        return JNI_TRUE;
    else
        return JNI_FALSE;
}

/* native code call java method */
JNIEXPRORT jboolean JNICALL Java_com_ex_Foo_callJavaFunc(JNIEnv *env, jobject obj, jint k)
{
    jboolean r = JNI_FALSE;
    jclass clazz = (*env)->GetObjectClass(env, obj);
    if (clazz == NULL)
        return r;
    jmethodID mID = (*env)->GetMethodID(env, clazz, "sayHello", "(I)Z");
    if (mID == NULL)
        return r;

    r = (*env)->CallBooleanMethod(env, obj, mID, 1);
    if (ExceptionCheck(env) == JNI_TRUE)
        return JNI_FALSE;
    else
        return r;
}

/**/
void fooPrintString(JNIEnv *env, jobject obj, jstring str)
{
    jchar *jstr = (*env)->GetStringUTFChars(env, str, 0);
    if (jstr == NULL)
        return NULL;

    printf("String from java:%s", jstr);
    (*env)->ReleaseStringUTFChars(env, str, jstr);
}

jint fooGetByteArray(JNIEnv *env, jclass clazz, jbyteArray array)
{
    jint len = (*env)->GetArrayLength(env, array);
    if (len < strlen("Hello Jni") + 1)
        return -1;

    jbyte *buf = (*env)->GetByteArrayElements(env, array, 0);
    if (buf == NULL)
        return -ENOMEM;

    jint r = sprintf(buf, "Hello Jni");
    (*env)->ReleaseByteArrayElements(env, array, buf, 0);
    return r;
}

static JNINativeMethod methods[] = {
    {"fooPrintString", "(Ljava/lang/String;)", (void *)fooPrintString},
    {"fooGetByteArray", "([B)I", (void *)fooGetByteArray},
};

jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
    JNIEnv *env;
    if ((*vm)->GetEnv(vm, (void **)&env, JNI_VERSION_1_4) != JNI_OK)
        return JNI_ERR;

    jclass clazz = (*env)->FindClass(env, "com/ex/Foo");
    if (clazz == NULL)
        return JNI_ERR;

    jint len = sizeof(methods) / sizeof(methods[0]);
    (*env)->RegisterNatives(env, clazz, methods, len);

    return JNI_VERSION_1_4;
}
```
