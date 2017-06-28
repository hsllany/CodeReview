Android JNI Base
=====

# JNINativeMethod

Defined in /libnativehelper/include/nativehelper/jni.h

```
typedef struct {
    const char* name;
    const char* signature;
    void*       fnPtr;
} JNINativeMethod;
```
