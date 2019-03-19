---
title: Android-Binder通信的数据传输载体-Parcel
date: 2019-03-19 14:09:19
tags:
	- Android
	- Binder
	- Parcel

category: Android
comments: ture
---

我们知道在 Android上常用的两种对象序列化工具是`Serializable`和`Parcelable`。尤其我们做和`Binder`相关的数据传输时，选择的都是`Parcelable`数据。`Parcelable`其实就是一个接口，约定，如果一个对象希望使用`Parcelable`方式将数据序列化到内存中，必须实现 `Parcelable`接口。  

```java
public void writeToParcel(Parcel dest, @WriteFlags int flags);
```

接口的一个核心方法就是 `writeToParcel`，其实就是调用`Parcel`提供的一系列方法，将数据序列化到`Parcel`中。   

#### Parcel

`Parcel`是一种数据的载体，用于承载Binder相关的通信数据，数据可以是原始类型的，也可以是对象的引用。  

`Parcel`常用的方法有:  

```java
writeByte(byte val) //写入一个byte
readByte()//读取一个byte
writeBoolean(boolean val)
readBoolean()
writeInt(int val)
readInt()
writeLong(long val)
readLong()
writeFloat(float val)
readFloat()
writeDouble(double val)
readDouble()
writeString(String val)
readString()
```

这些方法都是成对匹配使用的，即在数据封装端，调用相应的writexxx方法，在解封端使用readxxx方法。  

以上列举的操作数据类型是原始数据类型，和String，还可以写入其他类型:  

```java
public final void writeValue(Object v) {
        if (v == null) {
            writeInt(VAL_NULL);
        } else if (v instanceof String) {
            writeInt(VAL_STRING);
            writeString((String) v);
        } else if (v instanceof Integer) {
            writeInt(VAL_INTEGER);
            writeInt((Integer) v);
        } else if (v instanceof Map) {
            writeInt(VAL_MAP);
            writeMap((Map) v);
        } else if (v instanceof Bundle) {
            // Must be before Parcelable
            writeInt(VAL_BUNDLE);
            writeBundle((Bundle) v);
        } else if (v instanceof PersistableBundle) {
            writeInt(VAL_PERSISTABLEBUNDLE);
            writePersistableBundle((PersistableBundle) v);
        } else if (v instanceof Parcelable) {
            // types will be written.
            writeInt(VAL_PARCELABLE);
            writeParcelable((Parcelable) v, 0);
        } else if (v instanceof Short) {
            writeInt(VAL_SHORT);
            writeInt(((Short) v).intValue());
        } else if (v instanceof Long) {
            writeInt(VAL_LONG);
            writeLong((Long) v);
        } else if (v instanceof Float) {
            writeInt(VAL_FLOAT);
            writeFloat((Float) v);
        } else if (v instanceof Double) {
            writeInt(VAL_DOUBLE);
            writeDouble((Double) v);
        } else if (v instanceof Boolean) {
            writeInt(VAL_BOOLEAN);
            writeInt((Boolean) v ? 1 : 0);
        } else if (v instanceof CharSequence) {
            // Must be after String
            writeInt(VAL_CHARSEQUENCE);
            writeCharSequence((CharSequence) v);
        } else if (v instanceof List) {
            writeInt(VAL_LIST);
            writeList((List) v);
        } else if (v instanceof SparseArray) {
            writeInt(VAL_SPARSEARRAY);
            writeSparseArray((SparseArray) v);
        } else if (v instanceof boolean[]) {
            writeInt(VAL_BOOLEANARRAY);
            writeBooleanArray((boolean[]) v);
        } else if (v instanceof byte[]) {
            writeInt(VAL_BYTEARRAY);
            writeByteArray((byte[]) v);
        } else if (v instanceof String[]) {
            writeInt(VAL_STRINGARRAY);
            writeStringArray((String[]) v);
        } else if (v instanceof CharSequence[]) {
            // Must be after String[] and before Object[]
            writeInt(VAL_CHARSEQUENCEARRAY);
            writeCharSequenceArray((CharSequence[]) v);
        } else if (v instanceof IBinder) {
            writeInt(VAL_IBINDER);
            writeStrongBinder((IBinder) v);
        } else if (v instanceof Parcelable[]) {
            writeInt(VAL_PARCELABLEARRAY);
            writeParcelableArray((Parcelable[]) v, 0);
        } else if (v instanceof int[]) {
            writeInt(VAL_INTARRAY);
            writeIntArray((int[]) v);
        } else if (v instanceof long[]) {
            writeInt(VAL_LONGARRAY);
            writeLongArray((long[]) v);
        } else if (v instanceof Byte) {
            writeInt(VAL_BYTE);
            writeInt((Byte) v);
        } else if (v instanceof Size) {
            writeInt(VAL_SIZE);
            writeSize((Size) v);
        } else if (v instanceof SizeF) {
            writeInt(VAL_SIZEF);
            writeSizeF((SizeF) v);
        } else if (v instanceof double[]) {
            writeInt(VAL_DOUBLEARRAY);
            writeDoubleArray((double[]) v);
        } else {
            Class<?> clazz = v.getClass();
            if (clazz.isArray() && clazz.getComponentType() == Object.class) {
                writeInt(VAL_OBJECTARRAY);
                writeArray((Object[]) v);
            } else if (v instanceof Serializable) {
                // Must be last
                writeInt(VAL_SERIALIZABLE);
                writeSerializable((Serializable) v);
            } else {
                throw new RuntimeException("Parcel: unable to marshal value " + v);
            }
        }
    }
```
可见`Parcel`还支持多种 `Object`类型。

但是大部分情况下，封装端写入的对象和 解封端获得的对象不是同一个了，而是将对象中的数据复制过去一份，相当于是新 clone 了一个对象。那有没办法使用原来的对象呢？  

答案是可以的，不过是间接的，这类对象我们称之为`Active Objects`。Parcel 写入的是他们的特殊标记引用。这类对象常见的有以下两种:  

1. Binder。Binder 是Android系统中IPC的核心通信机制，它还是一个对象，Parcel 可以通过`writeStrongBinder(IBinder val)`写入Binder对象，解封端读取的是原Binder对象的一个特殊代理类(BinderProxy),但是最终的操作还是被原Binder对象响应，所以可以间接的认为将Binder对象传递了过去。  
2. FileDescriptor 。FD 是Linux中的文件描述符，可以通过Parcel 的`writeFileDescriptor(FileDescriptor val)`方法写入

不管是 Binder 还是 FileDescriptor 接收端的对象仍然会基于和原对象相同的操作，所以可以认为是 `Active Object`。  

#### Parcel 写入数据分析

我们基于 `Parcel`写入`String` 来做一个源码分析  

```java
public final void writeString(String val) {
        mReadWriteHelper.writeString(this, val);
    }
    
  public static class ReadWriteHelper {
        public static final ReadWriteHelper DEFAULT = new ReadWriteHelper();

        public void writeString(Parcel p, String s) {
            nativeWriteString(p.mNativePtr, s);
        }

        public String readString(Parcel p) {
            return nativeReadString(p.mNativePtr);
        }
    }
```

可见 `Parcel` 有 `native` 代码的实现  

```java
private Parcel(long nativePtr) {
        if (DEBUG_RECYCLE) {
            mStack = new RuntimeException();
        }
        init(nativePtr);
    }
    
private void init(long nativePtr) {
        if (nativePtr != 0) {
            mNativePtr = nativePtr;
            mOwnsNativeParcelObject = false;
        } else {
            mNativePtr = nativeCreate();//调用native代码，获得指针
            mOwnsNativeParcelObject = true;
        }
    }
```
`nativeCreate()`定义在`framework/base/cor/jni/android_os_Parcel.cpp`文件中  

```c++
static const JNINativeMethod gParcelMethods[] = {
    ...
   
    {"nativeWriteInt",            "(II)V", (void*)android_os_Parcel_writeInt},
    {"nativeWriteString",         "(ILjava/lang/String;)V", (void*)android_os_Parcel_writeString},
    {"nativeWriteStrongBinder",   "(ILandroid/os/IBinder;)V", (void*)android_os_Parcel_writeStrongBinder},
    {"nativeWriteFileDescriptor", "(ILjava/io/FileDescriptor;)V", (void*)android_os_Parcel_writeFileDescriptor},

    ...
    {"nativeReadInt",             "(I)I", (void*)android_os_Parcel_readInt},
    {"nativeReadLong",            "(I)J", (void*)android_os_Parcel_readLong},
    

    {"nativeCreate",              "()I", (void*)android_os_Parcel_create},
    ...
};
```

在`android_os_Parcel.cpp`中定义了native和java的方法映射。所以native层执行的方法是`android_os_Parcel_create()`  

```c++
static jint android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jint>(parcel);
}
```

所以Java层的`mNativePtr`变量实际上是native层的一个`Parcel(C++)`对象  

`Parcel.cpp`代码位置`frameworks/native/libs/binder/Parcel.cpp`
从源码位置上就能看出来，`Parcel`就是用来辅助Binder通信的。  

```c++
Parcel::Parcel()
{
    initState();
}

void Parcel::initState()
{
    mError = NO_ERROR;
    mData = 0;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    ALOGV("initState Setting data size of %p to %d\n", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %d\n", this, mDataPos);
    mObjects = NULL;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mHasFds = false;
    mFdsKnown = true;
    mAllowFds = true;
    mOwner = NULL;
}
```

Parcel 对象初始化的过程中只是简单的初始化了各个成员变量，并没有分配内存。Parcel 遵循的是动态扩展的原则，只有在真正需要的时候才会申请内存，避免了资源浪费。  

Parcel对象初始化的这些变量定义在``frameworks/native/libs/binder/Parcel.h`文件中，我们来看看它们的含义  

```c++
 	status_t            mError;
    uint8_t*            mData;//Parcel中存储的数据内容，它是一个8位 uint8_t类型的指针
    size_t              mDataSize;//Parcel 中已经存储的数据大小
    size_t              mDataCapacity;//最大容量
    mutable size_t      mDataPos;//当前数据存储到哪了？
    ...
```

我们看看 如何往`Parcel`中写入数据，以 写入String 为例:  

```java
 public final void writeString(String val) {
        mReadWriteHelper.writeString(this, val);
    }
 public void writeString(Parcel p, String s) {
            nativeWriteString(p.mNativePtr, s);
        }
```

执行的是`android_os_Parcel.cpp`中的`android_os_Parcel_writeString`方法  

```c++
static void android_os_Parcel_writeString(JNIEnv* env, jclass clazz, jint nativePtr, jstring val)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        status_t err = NO_MEMORY;
        if (val) {
            const jchar* str = env->GetStringCritical(val, 0);
            if (str) {
                err = parcel->writeString16(str, env->GetStringLength(val));
                env->ReleaseStringCritical(val, str);
            }
        } else {
            err = parcel->writeString16(NULL, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

首先将指针`mNativePtr`转成Native 层的Parcel对象；然后将Java成的String转成Native层的字符串；调用 Parcel对象的`writeString16 `方法入,这时候已经进入 `Parcel.cpp`中，我们看看 它的`writeString16 `方法    

```c++
status_t Parcel::writeString16(const String16& str)
{
    return writeString16(str.string(), str.size());
}

status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == NULL) return writeInt32(-1);
    
    status_t err = writeInt32(len);//写入数据长度
    if (err == NO_ERROR) {
        len *= sizeof(char16_t);//len * 单位大小 得到占用的空间大小
        uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
        if (data) {
            memcpy(data, str, len);
            *reinterpret_cast<char16_t*>(data+len) = 0;
            return NO_ERROR;
        }
        err = mError;
    }
    return err;
}
```

`writeString16 `方法首先通过`writeInt32 `写入字符串的长度  

```c++
status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}

//模板类
template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {//如果容量够
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;//保存val
        return finishWrite(sizeof(val));//修正mDataPos和mDataSize
    }

    status_t err = growData(sizeof(val));//数据超过Parcel的存储容量，需要扩容
    if (err == NO_ERROR) goto restart_write;
    return err;
}
```

写入String size 调用的是一个模板类，泛型T是int类型，首先判断是否容量充足，如果充足直接写入，否则扩容。  

回到 `writeString16 `方法，继续写入一个 String 字符串。`len *= sizeof(char16_t);`计算一共需要占用多少字节,然后使用`writeInplace `计算复制数据的目标地址，如果一切顺利，后面会调用`memcpy `真正的将字符串拷贝进去。  

我们分析一下`writeInplace `  

```c++
void* Parcel::writeInplace(size_t len)
{
    const size_t padded = PAD_SIZE(len);

    // sanity check for integer overflow
    if (mDataPos+padded < mDataPos) {
        return NULL;
    }

    if ((mDataPos+padded) <= mDataCapacity) {
restart_write:
        uint8_t* const data = mData+mDataPos;

        // Need to pad at end?
        if (padded != len) {//需要尾部填充
#if BYTE_ORDER == BIG_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0xffffff00, 0xffff0000, 0xff000000
            };
#endif
#if BYTE_ORDER == LITTLE_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0x00ffffff, 0x0000ffff, 0x000000ff
            };
#endif
          
            *reinterpret_cast<uint32_t*>(data+padded-4) &= mask[padded-len];//填充尾部
        }

        finishWrite(padded);//修正mDataPos和mDataSize
        return data;
    }

    status_t err = growData(padded);//容量不足，扩容
    if (err == NO_ERROR) goto restart_write;
    return NULL;
}
```

```
#define PAD_SIZE(s) (((s)+3)&~3)
```
`PAD_SIZE `以4字节对齐，即每一份占4字节。如

```  
len =1 padded = 4 
len =4 padded = 4 
len =5 padded = 8
len =8 padded = 8
len =9 padded = 12
```

回到`writeInplace `方法，如果 `padded != len`说明写入的数据长度不是4字节的整数倍，所以需要在空余的尾部填充一些 `mask`数据，这里填充的字节序有两种:`BIG_ENDIAN `和`LITTLE_ENDIAN `,他们的填充字段不同。

该方法中同样考虑了 容量不足时使用`growData `方法扩容。 

再次回到`writeString16 `方法中   

```
 if (data) {
            memcpy(data, str, len);
            *reinterpret_cast<char16_t*>(data+len) = 0;
            return NO_ERROR;
        }

```

有了拷贝的目标地址，然后系统调用`memcpy `将 字符串 `str`拷贝到目标地址中；
在最后写入结束标记`0`。 至此，写入一个String 操作完成。  

总结一下写入String 操作:  
1. Java成的`Parcel.writeString`最终调用`Parcel.cpp`中的`writeString16`方法
2. 首先写入一个4直接的Int值，表示 String的长度
3. 通过 `writeInplace`函数计算 要写入的目标地址，注意，这里以4字节为单位进行分配
4. 如果空间不足，使用函数`growData`进行扩容，中间会设计到调用`continueWrite`函数做老数据拷贝
5. 获得目标地址后，使用`memcpy`函数进行拷贝
6. 最后写入结束标记`0`

#### Parcel 读数据分析

上面分析了往Parcel 中写一个String，这里配对分析一下`readString`  
readString 也是经过natice 层处理的，最终调用到`android_os_Parcel.cpp`中  

```c++
static jstring android_os_Parcel_readString(JNIEnv* env, jclass clazz, jint nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        size_t len;
        const char16_t* str = parcel->readString16Inplace(&len);
        if (str) {
            return env->NewString(str, len);
        }
        return NULL;
    }
    return NULL;
}
```

又调用了`Parcel.cpp`中的`readString16Inplace `方法  

```c++
const char16_t* Parcel::readString16Inplace(size_t* outLen) const
{
    int32_t size = readInt32();//首先读出String的大小
    // watch for potential int overflow from size+1
    if (size >= 0 && size < INT32_MAX) {
        *outLen = size;
        const char16_t* str = (const char16_t*)readInplace((size+1)*sizeof(char16_t));//
        if (str != NULL) {
            return str;
        }
    }
    *outLen = 0;
    return NULL;
}
```

首先 readInt，读出String 的大小；然后通过`readInplace `获取到字符串的起始地址；
最后 new 一个Java层的字符串对象返回。

这里以 String的写入和读取为例做的分析，其他类型数据的写入和读取也都类似，就不一一分析了。至于 `Active Object`的写入和读取，在之后的Binder 分析文章中会有具体分析。



















