---
title: Android-智能指针
date: 2019-03-16 22:12:49
tags:
	- Android
	- Binder
	
category: Android
comments: ture
---

### 什么是智能指针?

Android 上层APP 基本都是基于Java 语言进行来发的，对于Java 语言来说是没有`指针`概念的，因为在 JVM层，Java 就将指针隐藏起来了，基于Java 开发不需要涉及对象的手动内存分配和释放。那Android 为什么搞一个 `智能指针`呢？ 这主要是因为 Android 的Framework 相当一部分还是 `C/C++`来实现的，`C/C++`是有指针概念的，但是对于一个庞大的操作系统而言，智能肯定是满天飞的，这中间很容易就出现指针相关的错误，为了减少在指针方面的关注，Android 封装了自己的`智能指针`用于管理`C/C++`层的对象引用问题，期望能像 Java一样，忘记指针的概念。  

但事实上，`智能指针`并不是真正的指针，它只是对`C++`对象自动回收机制的封装。

我们来看看智能指针如何实现的?  

#### 常见的指针问题

在 C/C++项目中，常见的指针问题大致有:  

- 指针未初始化
- new 操作和delete 操作未配套操作
- 野指针

Android的智能指针主要也是为了解决这三个问题  

__指针未初始化:__ 解决这个问题 只需要在创建智能指针的时候，将 指针置空  
__new 和 delete不配套:__ 在C++中，`构造函数`和`析构函数`是在对象new出来和delete的时候调用的,可以在这两个地方做相应的处理  
__野指针:__ 解决这个问题其实就是让智能指针自动能判断对象是否可以被回收。这也是三个问题中的核心问题。

智能指针采用的是`引用计数器`方式来比较一个对象是否需要被回收。  

_那么引用计数器是应该智能指针拥有呢还是被对象Object拥有呢？_  

如果计数器由智能指针拥有，假如 智能指针`SmartPointer1`和智能指针`SmartPointer2`都引用 object对象，当 `SmartPointer1`不再引用object时，计数器为0，删除了对象object，但是 `SmartPoint2`还在引用着object。回收object 肯定是不合理的。所以 引用计数器只能是 object 拥有。


### 智能指针的实现

智能指针在Android上有三个比较重要的实现，分别是`轻量级指针`,`强指针`,`弱指针`。它们的实现原理都是一致的，即由对象本身来提供引用计数器，但是它不会去维护这个引用计数器的值，而是由智能指针来维护

#### 轻量级智能指针 LightRefBase

这里所说的轻量级指针不是说 `LightRefBase` 是一个指针的定义，它是 配合智能指针的object对象的最简单，最轻量级的实现。  

源码位置 `frameworks/native/include/utils/RefBase.h`  

```c++
template <class T>
class LightRefBase
{
public:
    inline LightRefBase() : mCount(0) { } //mCount 就是引用计数器,初始化值为0
    inline void incStrong(__attribute__((unused)) const void* id) const {
        android_atomic_inc(&mCount);
    }
    inline void decStrong(__attribute__((unused)) const void* id) const {
        if (android_atomic_dec(&mCount) == 1) {
            delete static_cast<const T*>(this);
        }
    }
...

protected:
    inline ~LightRefBase() { }
...

private:
    mutable volatile int32_t mCount;
};
```
给类有一个成员变量`mCount`，这就是`引用计数器`，它的初始化值为`0`，另外，这个类还提供两个成员函数`incStrong`和`decStrong`,分别是用来增加引用和减少引用计数的，这两个函数提供给智能指针来调用，在decStrong函数中，如果当前引用计数值为1，那么当减1后就会变成0，于是就会delete这个对象。

轻量级智能指针 LightRefBase 是需要和 智能指针搭配使用的，只有继承 `LightRefBase `的类型对象才能使用智能指针(先不考虑RefBase)。  

#### 强指针 sp

前面说的 `LightRefBase `的计数器是由智能指针来修改的，那就是 `sp`了。  

源码定义位置：`frameworks/native/include/utils/StrongPointer.h`  

```c++
template <typename T>
class sp
{
public:
    inline sp() : m_ptr(0) { }//首先将引用置空，解决指针为初始化问题

    sp(T* other);
    sp(const sp<T>& other);
    template<typename U> sp(U* other);
    template<typename U> sp(const sp<U>& other);

    ~sp();//析构函数

    // Assignment

    sp& operator = (T* other);
    sp& operator = (const sp<T>& other);

    template<typename U> sp& operator = (const sp<U>& other);
    template<typename U> sp& operator = (U* other);

    //! Special optimization for use by ProcessState (and nobody else).
    void force_set(T* other);

    // Reset

    void clear();

    // Accessors

    inline  T&      operator* () const  { return *m_ptr; }
    inline  T*      operator-> () const { return m_ptr;  }
    inline  T*      get() const         { return m_ptr; }

    // Operators

    COMPARE(==)
    COMPARE(!=)
    COMPARE(>)
    COMPARE(<)
    COMPARE(<=)
    COMPARE(>=)

private:
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;
    void set_pointer(T* ptr);
    T* m_ptr;
};
```

强指针sp 初始化的时候 首先将 引用 `m_ptr `置空，`m_ptr `指向的就是`LightRefBase`,当然也可以是`RefBase`,只不过 `RefBase`还支持弱指针访问，比较复杂，后面会再详细介绍。  

先看看 sp 构造函数做了什么事情  

```
template<typename T>
sp<T>::sp(T* other)
: m_ptr(other)
  {
    if (other) other->incStrong(this);
  }
```

sp的构造函数有多个，这里只分析了其中一个，其他的类似。通过构造方法与引用对象关联时，首先调用了`LightRefBase`或`RefBase`的`incStrong `增加引用数量。  

sp 还重载了运算符`=`,当使用 `=`将一个对象与智能指针sp关联时，会操作object对象的引用数。  

```c++
template<typename T>
sp<T>& sp<T>::operator = (T* other)
{
    if (other) other->incStrong(this);
    if (m_ptr) m_ptr->decStrong(this);
    m_ptr = other;
    return *this;
}
```

当使用`=`运算符时，将判断该指针之前是否引用过别的对象，即 `m_ptr`是否不为空，如果有引用，`m_ptr`不为空，需要先将原引用对象的引用计数器减小，在将新引用的对象计数器增加，并和 `m_ptr`关联。  

sp的析构函数  

```c++
template<typename T>
sp<T>::~sp()
{
    if (m_ptr) m_ptr->decStrong(this);
}
```

当执行析构函数的时候，将减小引用对象的引用计数器。  

回顾上面提及的`LightRefBase`和`sp`好像无法避免循环引用的问题。而`弱指针wp`就是专门解决该问题的。  

`LightRefBase `之所以称之为 `轻量级的`，是与`RefBase`相比而言的。`RefBase `的实现比`LightRefBase `复杂许多，其内部包含了 弱指针wp的定义和实现。  

#### 弱指针wp

如前面所说，弱指针的主要使命就是解决循环引用的问题，我们看看它和 强指针sp的区别  

弱指针wp源码定义在 `frameworks/native/include/utils/RefBase.h`中  

```
template <typename T>
class wp
{
public:
    typedef typename RefBase::weakref_type weakref_type;
    //约定 弱指针执行的对象类型必须是 RefBase 类型，而不能是 LightRefBase
    
    inline wp() : m_ptr(0) { }

    wp(T* other);//构造函数
    wp(const wp<T>& other);
    wp(const sp<T>& other);
    ...

    ~wp();
    
    wp& operator = (T* other);//运算符重载
    wp& operator = (const wp<T>& other);
    wp& operator = (const sp<T>& other);
    ...
    
    void set_object_and_refs(T* other, weakref_type* refs);

    // promotion to sp
    
    sp<T> promote() const; // 将弱指针升级为强指针sp

	...

    // Operators
    //以下是重载一些逻辑运算符

    COMPARE_WEAK(==)
    COMPARE_WEAK(!=)
    COMPARE_WEAK(>)
    COMPARE_WEAK(<)
    COMPARE_WEAK(<=)
    COMPARE_WEAK(>=)

    inline bool operator == (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) && (m_refs == o.m_refs);
    }
    ...

private:
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;

    T*              m_ptr; //指向目标对象的指针
    weakref_type*   m_refs;//指向weakref_type类型的弱引用
};
```
和 强指针sp相比，wp有以下重要区别:  

- 除了指向目标对象的 m_ptr 外，wp另外还有一个 m_refs 指针，类型为 weakref_type
- 没有重载 ->, * 等运算符
- 有一个 promote()方法用来将wp提升为sp
- 弱指针的目标对象类型不是`LightRefBase`，而是 `RefBase`

惯例，我们看看 wp的构造函数实现:  

```
template<typename T>
wp<T>::wp(T* other)
    : m_ptr(other)
{
    if (other) m_refs = other->createWeak(this);
}
```

可以看出和 sp的构造方法不同，没有直接调用 目标对象的 `incStrong`方法，而是调用了`createWeak`方法。可见wp并没有直接增加目标对象的引用计数器，因为 `createWeak `方法是 `RefBase `中定义的，我们稍后会详细分析 `RefBase`  

析构函数:  

```
template<typename T>
wp<T>::~wp()
{
    if (m_ptr) m_refs->decWeak(this);
}
```

析构函数也没有直接操作 引用计数器。

我们再看看运算符`=`的重载  

```
template<typename T>
wp<T>& wp<T>::operator = (T* other)
{
    weakref_type* newRefs =
        other ? other->createWeak(this) : 0;
    if (m_ptr) m_refs->decWeak(this);
    m_ptr = other;
    m_refs = newRefs;
    return *this;
}
```

可以看出，运算符`=`同样也没有直接操作引用计数器，而是操作 `weakref_type`

综上，wp主要操作的是 `weakref_type`。并不直接影响引用计数器，所以称之为弱指针，不会改变引用关系。

#### RefBase

前面多处提到`RefBase `，我们可以认为 `RefBase`是既支持强指针sp的也支持弱指针wp的对象类型，如果你定义的对象需要用到弱指针，请务必继承`RefBase `,而如果只使用强指针，你可以选择轻量级的`LightRefBase`也可以选择适应大多数情况的`RefBase`

即`RefBase`不仅支持强指针还支持弱指针，它的源码定义位置: `frameworks/native/include/utils/RefBase.h`  
源码实现位置: `frameworks/native/libs/utils/RefBase.cpp`  

```c++
class RefBase
{
public:
            void            incStrong(const void* id) const;//增加强引用计数器，强指针sp专用
            void            decStrong(const void* id) const;//减少强引用计数器，强指针sp专用
    			....
    		//嵌套在 RefBase 内部，wp 中会操作该类型对象
    class weakref_type
    {
    public:
        RefBase*            refBase() const;
        
        void                incWeak(const void* id);//增加弱引用计数器
        void                decWeak(const void* id);//减少弱引用计数器
        ...
    };
    			// 用于创建一个weakref_type类型的对象，来处理弱引用关系
            weakref_type*   createWeak(const void* id) const;
            
            weakref_type*   getWeakRefs() const;
				...

    typedef RefBase basetype;

protected:
                            RefBase();//RefBase的构造函数
    virtual                 ~RefBase(); //RefBase的析构函数

    //! Flags for extendObjectLifetime()
    // 这些 Flag 参数 用来标记object的生命周期的，在 extendObjectLifetime 方法使用
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
    
            void            extendObjectLifetime(int32_t mode);
            
private:
    friend class weakref_type;
    class weakref_impl;
    
                            RefBase(const RefBase& o);
            RefBase&        operator=(const RefBase& o);

private:
		...
        weakref_impl* const mRefs;//真实的处理引用关系的成员变量，不同于LightRefBase 的 Int类型的 mCount
};
```

通过分析 `RefBase`以及 `weakref_type` 的定义,发现 `RefBase`和`LightRefBase`的区别，虽然 `RefBase`和`LightRefBase`都提供了`incStrong`和`decStrong`成员函数来操作它的引用计数器,但 `RefBase`它不像`LightRefBase`类一样直接提供一个整型值（mutable volatile int32_t mCount）来维护对象的引用计数，前面我们说过，复杂的引用计数技术同时支持强引用计数和弱引用计数，在`RefBase`类中，这两种计数功能是通过其成员变量`mRefs`来提供的。  

`mRefs`是`weakref_impl` 类型的,在`RefBase.h`文件中只定义了`weakref_impl `,具体实现在 `frameworks/native/libs/utils/RefBase.cpp` 中，在 RefBase.cpp 文件中还具体实现了`incStrong `,`decStrong `等方法，让我们一起来看看。   

首先我们分段分析一下 `mRefs `的`weakref_impl`类型实现。  

```c++
class RefBase::weakref_impl : public RefBase::weakref_type
{//继承 weakref_type
public:
    volatile int32_t    mStrong;//强引用计数器
    volatile int32_t    mWeak;//弱引用计数器
    RefBase* const      mBase;
    volatile int32_t    mFlags;

#if !DEBUG_REFS //宏DEBUG_REFS来区分DEBUG环境和正式环境

    weakref_impl(RefBase* base)
        : mStrong(INITIAL_STRONG_VALUE)
        , mWeak(0)
        , mBase(base)
        , mFlags(0)
    {
    }

    void addStrongRef(const void* /*id*/) { }
    void removeStrongRef(const void* /*id*/) { }
    ...
#else // DEBUG环境下

    weakref_impl(RefBase* base)
        : mStrong(INITIAL_STRONG_VALUE)
        , mWeak(0)
        , mBase(base)
        , mFlags(0)
        , mStrongRefs(NULL)
        , mWeakRefs(NULL)
        , mTrackEnabled(!!DEBUG_REFS_ENABLED_BY_DEFAULT)
        , mRetain(false)
    {
    }
    
    ~weakref_impl()//weakref_impl析构函数
    {
    ... 省略...
    }

    void addStrongRef(const void* id) {
       //增加强引用计数器，这里使用了 mStrongRefs指针
        addRef(&mStrongRefs, id, mStrong);
    }

   ...

    void addWeakRef(const void* id) {
        addRef(&mWeakRefs, id, mWeak);
    }

    ...

private:
    struct ref_entry //结构体 ref_entry
    {
        ref_entry* next;
        const void* id;
#if DEBUG_REFS_CALLSTACK_ENABLED
        CallStack stack;
#endif
        int32_t ref;
    };

    void addRef(ref_entry** refs, const void* id, int32_t mRef)
    {
		...//省略
    }

    void removeRef(ref_entry** refs, const void* id)
    {
       ...//省略
    }

    ...

    mutable Mutex mMutex;
    ref_entry* mStrongRefs;//用来做强引用记录
    ref_entry* mWeakRefs; //弱引用记录
		...
#endif
};
```

通过查看源码实现，我们知道 `RefBase`不同于`LightRefBase`,它使用结构体`ref_entry `来记录引用关系。成员变量`mStrong `做强引用计数器，`mWeak `做弱引用计数器。  

回到 前面 wp 构造函数和 `=`运算符重载中，都调用了`RefBase.createWeak()`方法用来生成一个 `weakref_type` 实例，其实就是 `weakref_impl`。  

```
RefBase::weakref_type* RefBase::createWeak(const void* id) const
{
    mRefs->incWeak(id);
    return mRefs;
}
```

在`RefBase.h`定义中，`mRefs`定义为 `weakref_impl` 类型,所以这里调用的是`weakref_impl中的incWeak()`方法  

mRefs 是在 RefBase对象初始化的时候就在构造方法中完成了赋值  

```
RefBase::RefBase()
    : mRefs(new weakref_impl(this))
{
}
```
所以 `mRefs`就是一个`weakref_impl` 实例。  当弱指针wp需要第一次初始化弱引用时，会直接返回 `mRefs `对象，并且 操作`mRefs `对象执行`incWeak()`方法，增加一次弱引用计数。  

```
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->addWeakRef(id);
    const int32_t c = android_atomic_inc(&impl->mWeak);
    ALOG_ASSERT(c >= 0, "incWeak called on %p after last weak ref", this);
}
```

果然，在`incWeak `方法中，调用了`weakref_impl`的`addWeakRef`，而 `addWeakRef`最终通过`addRef`实现:  

```
void addRef(ref_entry** refs, const void* id, int32_t mRef)
    {
        if (mTrackEnabled) {
            AutoMutex _l(mMutex);

            ref_entry* ref = new ref_entry;
            // Reference count at the time of the snapshot, but before the
            // update.  Positive value means we increment, negative--we
            // decrement the reference count.
            ref->ref = mRef;
            ref->id = id;
#if DEBUG_REFS_CALLSTACK_ENABLED
            ref->stack.update(2);
#endif
            ref->next = *refs;
            *refs = ref;
        }
    }
```

其实就是新创建一个ref_entry，把 弱引用指针 wp本身 包装在 一个 ref_entry 中，添加到单链表的ref_entry 数据结构中，建立弱引用链表。  
将wp添加到弱引用链表之后，`incWeak `方法还通过`android_atomic_inc `增加了弱引用计数器`mWeak`的值。  

到目前为止，wp如何实现弱引用计数已经分析完成，总结一下:  

1. wp 通过构造方法或重载运算符= 会调用 RefBase的createWeak方法，来创建一个weakref_type 也就是weakref_impl实例
2. weakref_impl 在RefBase对象构造的时候就已经创建好了，将 weakref_impl 对象返回给 wp
3. 触发weakref_impl的incWeak 方法，将wp指针添加到弱引用链表中，并使用Android的原子操作方法，增加弱引用计数器 mWeak的值。

前面说了，RefBase 不仅支持wp还像 LightRefBase 一样支持 强指针sp。  

```c++
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->incWeak(id);//增加弱引用计数器
    
    refs->addStrongRef(id);
    const int32_t c = android_atomic_inc(&refs->mStrong);//增加强引用计数器，返回的是增加之前的值
	... 省略log
    if (c != INITIAL_STRONG_VALUE)  {//判断是不是第一次被引用
        return;
    }

	//如果是第一次被引用，需要调整一下 mStrong的值
    android_atomic_add(-INITIAL_STRONG_VALUE, &refs->mStrong);
    refs->mBase->onFirstRef();//是第一次被引用时，会回调 RefBase的onFirstRef()方法
}
```

通过源码看出，RefBase的强引用增加引用时，会同时增加弱引用计数器。还会判断是否是第一次被引用。如果是第一次被引用需要修改强引用计数器`weakref_impl `中`mStrong `的值；  
因为`weakref_impl `初始化的时候，`mStrong`被赋值为`INITIAL_STRONG_VALUE`  

```
#define INITIAL_STRONG_VALUE (1<<28)
```

`android_atomic_inc`方法是在 原 mStrong 的基础上加一，需要再减去 原始值`INITIAL_STRONG_VALUE`。  

下面我们在分析一下 RefBase对象什么时候被释放  

```c++
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id);
    const int32_t c = android_atomic_dec(&refs->mStrong);
    ... //省略 log
    if (c == 1) {//当引用数字当前是1时，会回调 RefBase的onLastStrongRef 方法
        refs->mBase->onLastStrongRef(id);
        if ((refs->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {//判断当前引用输入强引用关系时，会释放 对象
            delete this;
        }
    }
    refs->decWeak(id);//正常执行减小弱引用计数器
}
```

首先减少强引用计数器mStrong,如果减少到0了(c ==1 ，因为c 返回的是上一次计数器的值)就执行 RefBase对象的 onLastStrongRef 方法，并且，如果当前引用类型是强引用，就释放引用的对象。  

因为`incStrong `中同步增加了弱引用计数器，同样，为了对称，`decStrong `时也应该调用`decWeak `。  

```c++
void RefBase::weakref_type::decWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->removeWeakRef(id);//将弱引用指针wp从 弱引用链表中移除
    const int32_t c = android_atomic_dec(&impl->mWeak);//减小弱引用计数器
    ALOG_ASSERT(c >= 1, "decWeak called on %p too many times", this);
    if (c != 1) return;

    if ((impl->mFlags&OBJECT_LIFETIME_WEAK) == OBJECT_LIFETIME_STRONG) {//该引用类型是强引用类型
        if (impl->mStrong == INITIAL_STRONG_VALUE) {
            //该对象从没有被强引用指针引用过，如果 弱引用都没有了，当然释放对象
            delete impl->mBase;
        } else {
           // 最后一个强引用也消失的时候，这里释放 weakref_impl 对象
            delete impl;
        }
    } else {//该引用类型是弱引用
        // less common case: lifetime is OBJECT_LIFETIME_{WEAK|FOREVER}
        impl->mBase->onLastWeakRef(id);
        if ((impl->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) {//最后一个弱引用消失时，释放对象
            delete impl->mBase;
        }
    }
}
```

将弱引用指针wp从弱引用链表中移除，减小 `mWeak` 计数器; 如果当时是最后一个弱引用执行dec操作，会根据 `LIFETIME` 做不同的处理，具体看上述源码注释。 

前面多次提到，前引用增加时会同步的增加弱引用计数器，但是弱引用增加时肯定不会增加强引用计数器，所以弱引用计数器的值一定会大于强引用计数器，让程序走到这里的时候，弱引用计数器一定为0，而强引用计数器值有两种可能，一种就是  `INITIAL_STRONG_VALUE `即从来没有强引用指针引用，这种情况当然要释放对象；第二种就是在有强引用的情况下，强引用的`decStrong` 方法会释放`RefBase`对象，在这里只需要释放`weakref_impl `对象就行了，不需要再重复释放 `RefBase`对象。

总结:  
1. Android中的智能指针分为`强指针sp`和`弱指针wp`
2. 有一个配合sp使用的`LightRefBase`，我们称之为`轻量级指针`
3. 通常和智能指针搭配使用的对象必须是 `RefBase` 类型，如果不需要 `弱指针wp`，可以使用轻量级的 `LightRefBase` 实现轻量级指针
4. 对于 `RefBase` 对象而言，增加强引用也会同步的增加一个弱引用，反之不会。
5. 弱指针要想访问对象，必须升级为强指针，通过弱指针wp的`promote`方法，最终调用`weakref_impl `的`attemptIncStrong`。


























