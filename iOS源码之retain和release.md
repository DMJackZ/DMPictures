Objective-C同时为我们提供了增加引用计数的 retain 和减少引用计数的 release 方法。  
这篇文章会在源码层面介绍 Objective-C 中 retain 和 release 的实现。  
##从retain开始  
如今我们已经进入全面使用ARC的时代，几年前还经常使用 retain 和 release 方法已经很难出现在我们的视野中了，绝大多数内存管理的实现细节都由编译器代劳了。  
```
- (id)retain {
    return ((id)self)->rootRetain();
}

ALWAYS_INLINE id 
objc_object::rootRetain()
{
    return rootRetain(false, false);
}

```
这个id objc_object::rootRetain(bool tryRetain, bool handleOverflow) 方法是最重要的方法，其原理就是将 isa 结构体中的 extra_rc 的值加一。  
extra_rc 就是用于保存自动引用计数的标志位。
下面是ARM64时 isa 结构体的结构
```
union isa_t
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
    
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33;
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
};
```  
接下来分三种情况对 rootRetain(bool tryRetain, bool handleOverflow) 方法进行分析。  
### 正常的rootRetain
下面是简化后的方法实现，只处理一般情况的代码：
```
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    return (id)this;
}
```  
> 这里假设条件是 isa 中的 extra_rc 的位数足以存储引用计数  
- 1、使用 LoadExclusive 加载 isa 的值
- 2、调用 addc(newisa.bits, RC_ONE, 0, &carry) 方法将 isa 中 extra_rc 的值加一
- 3、调用 StoreExclusive(&isa.bits, oldisa.bits, newisa.bits) 更新 isa 的值
- 4、返回当前对象  
### 进位版本的rootRetain  
这里调用 addc 方法将 isa 中的 extra_rc 值加一时， extra_rc 不足以保存引用计数，引起进位时。
```
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    isa_t oldisa;
    isa_t newisa;

    oldisa = LoadExclusive(&isa.bits);
    newisa = oldisa;

    uintptr_t carry;
    newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

    if (slowpath(carry)) {
       // newisa.extra_rc++ overflowed
       if (!handleOverflow) {
           ClearExclusive(&isa.bits);
           return rootRetain_overflow(tryRetain);
       }
    }
}
```
> extra_rc 不足以保存引用计数，并且 handleOverflow 为 false。  
当传入的 handleOverflow 为 false 时，我们会调用 rootRetain_overflow 方法
```
NEVER_INLINE id 
objc_object::rootRetain_overflow(bool tryRetain)
{
    return rootRetain(tryRetain, true);
}
```
> 这个方法就是重新执行 rootRetain 方法，并传入 handleOverflow 为 true。
### 进位版本的rootRetain 处理溢出
当传入的 handleOverflow = true 时， 会在 rootRetain 方法中处理引用计数的溢出。
```
ALWAYS_INLINE id
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (slowpath(carry)) {
            // newisa.extra_rc++ overflowed
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    if (slowpath(transcribeToSideTable)) {
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    return (id)this;
}
```
因为 extra_rc 已经溢出了，更新它的值为 RC_HALF:
```
define RC_HALF  (1ULL<<18)
```
> extra_rc 为19位， RC_HALF 为 1 左移18位  

之后设置 has_sidetable_rc 为 true，存储新的 isa 值之后，调用 sidetable_addExtraRC_nolock 方法
```
bool 
objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}
```
这里将溢出的 RC_HALF 存储到 SideTable  
在iOS的内存管理中，使用了 isa 结构体中的 extra_rc 和 SideTable 来存储对象的自动引用计数.
## 以release结束
```
- (oneway void)release {
    ((id)self)->rootRelease();
}
ALWAYS_INLINE bool 
objc_object::rootRelease()
{
    return rootRelease(true, false);
}

```
在分析 release 方法时，根据上下文的不同，将 release 方法的实现拆分为三部分。
### 正常的 release
```
ALWAYS_INLINE bool
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
       
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
    } while (slowpath(!StoreReleaseExclusive(&isa.bits,
                                             oldisa.bits, newisa.bits)));

    return false;
}
```
- 1、使用 LoadExclusive 获取 isa 内容
- 2、调用 subc 将 isa 中 extra_rc 引用计数减一
- 3、调用 StoreReleaseExclusive 方法保存新的 isa
### 从 SideTable 借位
```
ALWAYS_INLINE bool
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
       
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // don't ClearExclusive()
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits,
                                             oldisa.bits, newisa.bits)));

    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate

    // abandon newisa to undo the decrement
    newisa = oldisa;

    if (slowpath(newisa.has_sidetable_rc)) {
        if (!handleUnderflow) {
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc);
        }
    }
}
```
> 需要借位，并且 handleUnderflow 为 false 时。

当传入的 handleUnderflow 为 false 时，会调用 rootRelease_underflow 方法
```
NEVER_INLINE bool 
objc_object::rootRelease_underflow(bool performDealloc)
{
    return rootRelease(performDealloc, true);
}
```
### 借位版本的 rootRelease 处理借位
当传入的 handleUnderflow 为 true时， 会在 rootRelease 中处理借位
```
ALWAYS_INLINE bool
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    bool sideTableLocked = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
       
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // don't ClearExclusive()
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits,
                                             oldisa.bits, newisa.bits)));

    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate

    // abandon newisa to undo the decrement
    newisa = oldisa;

    if (slowpath(newisa.has_sidetable_rc)) {
        if (!handleUnderflow) {
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc);
        }

        // Try to remove some retain counts from the side table.
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF);

        // To avoid races, has_sidetable_rc must remain set
        // even if the side table count is now zero.

        if (borrowed > 0) {
            // Side table retain count decreased.
            // Try to add them to the inline count.
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits,
                                                oldisa.bits, newisa.bits);
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }
}
```
- 借位时会调用sidetable_subExtraRC_nolock函数  
	- 借位成功时函数返回 RC_HALF，更新 extra_rc 的值为 RC_HALF - 1 ，调用 StoreReleaseExclusive 更新 isa的值
	- 借位失败时函数返回 0， 也就是对象需要销毁
函数sidetable_subExtraRC_nolock如下，传入参数为RC_HALF：
```
size_t 
objc_object::sidetable_subExtraRC_nolock(size_t delta_rc)
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()  ||  it->second == 0) {
        // Side table retain count is zero. Can't borrow.
        return 0;
    }
    size_t oldRefcnt = it->second;

    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    size_t newRefcnt = oldRefcnt - (delta_rc << SIDE_TABLE_RC_SHIFT);
    assert(oldRefcnt > newRefcnt);  // shouldn't underflow
    it->second = newRefcnt;
    return delta_rc;
}
```
- 从SideTable 中借位时 
	- SideTable 中 retain count 为0 时，借位失败函数返回0

	- SideTable 中 retain count 不为0 时，借位成功函数返回RC_HALF
### rootRelease 中调用 dealloc
```
// Really deallocate.

    if (slowpath(newisa.deallocating)) {
        ClearExclusive(&isa.bits);
        if (sideTableLocked) sidetable_unlock();
        return overrelease_error();
        // does not actually return
    }
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;

    if (slowpath(sideTableLocked)) sidetable_unlock();

    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
```
> 这段代码是 处理借位时 borrowed 为0 时，对象的 dealloc 处理
直接调用objc_msgSend向当前对象发送 SEL_dealloc 消息， 为了确保消息只发送一次，使用了 deallocating 标记

## 获取 retainCount
```
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}

inline uintptr_t 
objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}

size_t 
objc_object::sidetable_getExtraRC_nolock()
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) return 0;
    else return it->second >> SIDE_TABLE_RC_SHIFT;
}

```
retainCount由三部分组成：
- 数值1
- extra_rc 中存储的值
- sidetable_getExtraRC_nolock 从sideTable中得到的值
## 小结
- Objective-C 使用 isa 中的 extra_rc 和 SideTable 来存储对象的引用计数
- 对象实际的引用计数会在 extra_rc 和 SideTable 的基础上加一
- 在对象的引用计数归零时，会通过 objc_msgSend 调用 SEL_dealloc 回收对象
