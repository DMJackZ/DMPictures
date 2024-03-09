# 这篇文章会在源代码层面介绍Objective-C中自动释放池，以及方法的autorelease的具体实现  #

# 从main函数开始 #
main 函数可以说是在整个iOS开发中非常不起眼的一个函数，却是整个iOS应用的入口。  
main.m 文件中的内容是这样的：
```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
在这个@autoreleasepool block中只包含了一行代码，这行代码将所有的事件、消息全部交给了UIApplication来处理，但是这不是本文关注的重点。  
需要注意的是：**整个iOS的应用都是包含在一个自动释放池block中的。**  
# @autoreleasepool  #
@autoreleasepool到底是什么？我们在命令行中使用clang-rewrite-objc main.m 让编译器重新改写这个文件：  
```
 clang -rewrite-objc main.m
```
当前目录下多了一个main.cpp文件  
```
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        appDelegateClassName = NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class")));
    }
    return UIApplicationMain(argc, argv, __null, appDelegateClassName);
}
```
>这里只看main函数中的代码  
>
在这个文件中，有一个非常奇怪的__AtAutoreleasePool 的结构体，前面的注释写到/* @autoreleasepool */ 。也就是说@autoreleasepool {} 被转换为一个 __AtAutoreleasePool 的结构体。  
想要弄清楚这行代码的意义，我们要在main.cpp 中查找名为 __AtAutoreleasePool 的结构体:  
```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```
这个结构体会在初始化时调用objc_autoreleasePoolPush()方法，会在析构时调用objc_autoreleasePoolPop方法。  
这表明，我们的main 函数在实际工作时其实是这样的：
```
int main(int argc, const char * argv[]) {
    {
        void *atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        // do whatever you want
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```
@autoreleasepool 只是帮助我们少写了这两行代码而已，让代码看起来更美观，然后要根据上述两个方法来分析自动释放池的实现。  
# Autoreleasepool是什么  #
这一节开始分析objc_autoreleasePoolPush 和 objc_autoreleasePoolPop 的实现，在 runtime源码的NSObject.mm文件中：  
```
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```
上面的方法看上去是对 AutoreleasePoolPage 对应的静态方法 push 和 pop 的封装。  
这一小节会按照下面的顺序逐步解析代码中的内容：  
- [AutoreleasePoolPage的结构](#1)
- [objc_autoreleasePoolPush方法](#2)
- [objc_autoreleasePoolPop方法](#3)
<a name="1"> </a>
## AutoreleasePoolPage的结构
AutoreleasePoolPage 是一个C++中的类，它在 NSObject.mm 中的定义是这样的：  
```
class AutoreleasePoolPage 
{
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
}
```
- magic 用于对当前 AutoreleasePoolPage 完整性的校验
- thread 保存了当前页所在的线程
- parent 指向父节点，第一个节点的parent值为nil
- child  指向子节点，最后一个节点的child值为nil

**每一个自动释放池都是由一系列的 AutoreleasePoolPage 组成的，并且每一个 AutoreleasePoolPage 的大小都是4096字节（16进制0x1000）**  

### 双向链表
自动释放池中的 AutoreleasePoolPage 是以**双向链表**的形式连接起来的。  
![双向链表2](https://github.com/DMJackZ/DMPictures/blob/main/AutoreleasePoolPageLink.png)  
> parent 和child 就是用来构造双向链表的指针。
>

### 自动释放池中的栈
如果我们的一个 AutoreleasePoolPage 被初始化在内存的 0x100816000 ~ 0x100817000 中， 其中有 56 Byte(字节)用于存储 AutoreleasePoolPage 的成员变量， 剩下的 0x100816038 ~ 0x100817000 都是用来存储加入到自动释放池中的对象。  
![内存布局](2.png) 
> AutoreleasePoolPage 的内存大小为56 字节， magic_t 结构体成员magic 占用内存为 uint32_t m[4], 即为 4 * 4 共16字节；属性next 、thread、parent、child 均占8个字节，共32字节；uint32_t 两个 depth 和 hiwat 各占4字节，共8字节。
```
id * begin() {
    return (id *) ((uint8_t *)this+sizeof(*this));
}

id * end() {
    return (id *) ((uint8_t *)this+SIZE);
}
```
- begin() 和 end() 这两个类的实例方法帮助我们快速获取 0x100816038 ~ 0x100817000 这一范围的边界地址。 
 
- next 指向了下一个为空的内存地址，如果 next 指向的地址加入了一个 object，它就会移动到下一个为空的内存地址中。  

### POOL_BOUNDARY （哨兵对象）
到了这里，你可能想知道 POOL_BOUNDARY 到底是什么，还有它为什么在栈中。  
首先回答第一个问题：POOL_BOUNDARY 只是 nil 的别名。
```
#   define POOL_BOUNDARY nil
```
在每个自动释放池初始化调用 objc_autoreleasePoolPush 的时候，都会把一个 POOL_BOUNDARY push 到自动释放池的栈顶，并且返回这个 POOL_BOUNDARY 哨兵对象。
```
int main(int argc, const char * argv[]) {
    {
        void *atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        // do whatever you want
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```
> 上面的 atautoreleasepoolobj 就是一个 POOL_BOUNDARY 。

而当方法 objc_autoreleasePoolPop 调用时，就会向自动释放池中的对象发送 release 消息，直到第一个 POOL_BOUNDARY 。

<a name="2"> </a>
## objc_autoreleasePoolPush 方法
我们来重新回顾一下 objc_autoreleasePoolPush 方法：
```
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```
它调用 AutoreleasePoolPage 的类方法 push，也非常简单:
```
static inline void *push()
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

根据DebugPoolAllocation 判断进入 autoreleaseNewPage 或者 autoreleaseFast，并传入哨兵对象 POOL_BOUNDARY:
```
static __attribute__((noinline))
id *autoreleaseNewPage(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page) return autoreleaseFullPage(obj, page);
    else return autoreleaseNoPage(obj);
}
```

在这里会进入一个比较关键的方法 autoreleaseFast，并传入哨兵对象 POOL_BOUNDARY:
```
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```
autoreleaseFast 方法分三种情况选择不同的代码执行：
- 有 hotPage 并且当前 page 不满时
	- 调用 page->add(obj) 方法将对象添加到 AutoreleasePoolPage 的栈中
- 有 hotPage 并且当前page 已满时
	- 调用 autoreleaseFullPage 初始化一个新的页
	- 调用 page->add(obj) 方法将对象添加到 AutoreleasePoolPage 的栈中
- 无 hotPage时
	- 调用 autoreleaseNoPage 穿件一个 hotPage
	- 调用 page->add(obj) 方法将对象添加到 AutoreleasePoolPage 的栈中
最后都会调用 page->add(obj) 将对象添加到自动释放池中。
> hotPage 可以理解为当前正在使用的 AutoreleasePoolPage 。

```
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full.
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);
    
    // 通过page的 child指针，获取下一个page对象，
    // 如果下一个page对象为空，则调用AutoreleasePoolPage创建一个新的page对象
    // while 循环判断page，直到page 没有装满
    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    // 将while循环得到的page对象设置为hotpage
    setHotPage(page);
    
    // 将obj对象加入到page中
    return page->add(obj);
}
```

autoreleaseFullPage 方法的代码执行：
- 1.传入的page必须是hotPage， 必须full。
- 2.通过child 指针，从当前page寻找，直到获取一个没有装满的page，或者创建一个新的page。
- 3.将通过while遍历获取到的page设置为hotPage，并且将obj加入到page中

```
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    assert(!hotPage());

    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
        // We are pushing a second pool over the empty placeholder pool
        // or pushing the first object into the empty placeholder pool.
        // Before doing that, push a pool boundary on behalf of the pool
        // that is currently represented by the empty placeholder.
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place,
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug",
                     pthread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        // We are pushing a pool with no pool in place,
        // and alloc-per-pool debugging was not requested.
        // Install and return the empty pool placeholder.
        return setEmptyPoolPlaceholder();
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
    
    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }
    
    // Push the requested object or pool.
    return page->add(obj);
}
```
autoreleaseNoPage 方法的代码执行:
- 1. 创建一个新的page
- 2. 将page设置为hotpage
- 3. 根据判断是否需要将 POOL_BOUNDARY 哨兵对象加入到page
- 4. 将obj加入到page中

```
id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```
page->add 添加对象
这个方法就是一个压栈的操作，将对象加入 AutoreleasePoolPage 然后移动栈顶的指针。
<a name="3"> </a>
## objc_autoreleasePoolPop 方法
```
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```
我们一般都会在这个方法中传入一个哨兵对象 POOL_BOUNDARY

```
static inline void pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;

    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        // Popping the top-level placeholder pool.
        if (hotPage()) {
            // Pool was used. Pop its contents normally.
            // Pool pages remain allocated for re-use as usual.
            pop(coldPage()->begin());
        } else {
            // Pool was never used. Clear the placeholder.
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not
            // fatal in executables built with old SDKs.
            return badPop(token);
        }
    }

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top)
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    }
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

该pop方法总共做了三件事:
- 1.使用 pageForPointer 获取 token 所在的 AutoreleasePoolPage
- 2.调用 releaseUntil 方法释放栈中的对象，直到 stop
- 3.调用 page 的 kill 方法
	- 当前page不为空时, 如果当前page没有child，就保留当前page，不调用当前page的kill
	- 当前page不为空时， 如果当前page有child
		- 当前page占用空间不超过一半，从child开始释放，也就是调用当前page->child->kill
		- 当前page占用空间超过一半，没有 孙子辈 page时，当前page的 child 也就是不释放了 （为了性能提升，少释放一个page的内存空间）；有孙子辈 page时，从孙子辈page 开始释放，也就是调用page->child->child->kill

### pageForPointer 获取 AutoreleasePoolPage
```
static AutoreleasePoolPage *pageForPointer(const void *p)
{
    return pageForPointer((uintptr_t)p);
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p)
{
    AutoreleasePoolPage *result;
    
    //SIZE: 4096,每页page的大小
    // p % SIZE : 获取p 在page中的偏移地址
    uintptr_t offset = p % SIZE;

    // 边界检测： 偏移地址 <= page 的内存大小
    assert(offset >= sizeof(AutoreleasePoolPage));

    //获取p所在page的起始地址
    result = (AutoreleasePoolPage *)(p - offset);
    
    result->fastcheck();

    return result;
}
```
pageForPointer 方法主要是通过内存地址的操作，获取当前指针所在页的首地址  
> 将指针与页面的大小，也就是4096 取模，得到当前指针的偏移量，因为所有的 AutoreleasePoolPage 在内存中都是对齐的。  
> 最后调用方法 fastcheck 来检查当前的 result 是不是一个 AutoreleasePoolPage

### releaseUntil 释放对象
```
void releaseUntil(id *stop)
{
    // Not recursive: we don't want to blow out the stack
    // if a thread accumulates a stupendous amount of garbage
    
    // while循环，直到next == stop
    while (this->next != stop) {
        // Restart from hotPage() every time, in case -release
        // autoreleased more objects
        AutoreleasePoolPage *page = hotPage();

        // fixme I think this `while` can be `if`, but I can't prove it
        // 如果page为空，利用parent指针找到一个不为空的page，并设置为hotpage
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        //将page所在的内存区域设置为可读可写
        page->unprotect();
        
        //通过next指针获取page中记录的对象，next -1 前移
        id obj = *--page->next;
        
        // void *memset(void *s, int ch, size_t n);
        // 将s中当前位置后面的n个字节 （typedef unsigned int size_t ）用 ch 替换并返回 s 。
        
        // 将当前对象的 8个字节 用0xA3 替换
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        
        // 修改完page中的值后，设置page所在的内存区域为只读
        page->protect();

        // 如果对象不是哨兵对象则释放
        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    // 把当前 page 设置 hotpage
    setHotPage(this);

#if DEBUG
    // we expect any children to be completely empty
    for (AutoreleasePoolPage *page = child; page; page = page->child) {
        assert(page->empty());
    }
#endif
}
```
用一个while 循环持续释放 AutoreleasePoolPage 中的内容，直到 next 指向了 stop 。  
使用 memset 将内存的内容设置成 SCRIBBLE （常量0xA3），然后使用 objc_release 释放对象内存。 
> releaseUntil 函数只是将page中的对象释放了，并且对应的位置用 SCRIBBLE （常量0xA3）填充，但是child/parent 指针没有清空，也就是说page还在内存中，没有释放。释放page的操作在后续kill完成。

### kill 方法
```
void kill()
{
    // Not recursive: we don't want to blow out the stack
    // if a thread accumulates a stupendous amount of garbage
    AutoreleasePoolPage *page = this;
    while (page->child) page = page->child;

    AutoreleasePoolPage *deathptr;
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->unprotect();
            page->child = nil;
            page->protect();
        }
        delete deathptr;
    } while (deathptr != this);
}
```
kill会将当前页面以及子页面全部删除,释放AutoreleasePoolPage占用空间，是从最尾部的子page开始释放

## autorelease 方法
```
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}

inline id 
objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}

__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}

```

```
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```
在autorelease 方法的调用栈中，最终都会调用到上面提到的 autoreleaseFast 方法，将当前对象加入到 AutoreleasePoolPage 中。 由于上面已经分析过 autoreleaseFast 方法的实现了，参考上面。

## 实战验证自动释放池内存结构
在 ARC 模式下，是无法手动调用 autorelease，所以要将项目切换至MRC模式 Build Settings -> Objective-C Automatic Reference Counting 设置为 NO，如下图所示：
![MRC模式]()

- 需要用到_objc_autoreleasePoolPrint
```
void 
_objc_autoreleasePoolPrint(void)
{
    AutoreleasePoolPage::printAll();
}

```
很简单，就是对 AutoreleasePoolPage 类方法 printAll 的调用
```
static void printAll()
{
    _objc_inform("##############");
    _objc_inform("AUTORELEASE POOLS for thread %p", pthread_self());

    AutoreleasePoolPage *page;
    ptrdiff_t objects = 0;
    for (page = coldPage(); page; page = page->child) {
        objects += page->next - page->begin();
    }
    _objc_inform("%llu releases pending.", (unsigned long long)objects);

    if (haveEmptyPoolPlaceholder()) {
        _objc_inform("[%p]  ................  PAGE (placeholder)",
                     EMPTY_POOL_PLACEHOLDER);
        _objc_inform("[%p]  ################  POOL (placeholder)",
                     EMPTY_POOL_PLACEHOLDER);
    }
    else {
        for (page = coldPage(); page; page = page->child) {
            page->print();
        }
    }

    _objc_inform("##############");
}
```
存在page的情况下，循环遍历page，调用page的print 方法
```
void print()
{
    _objc_inform("[%p]  ................  PAGE %s %s %s", this,
                 full() ? "(full)" : "",
                 this == hotPage() ? "(hot)" : "",
                 this == coldPage() ? "(cold)" : "");
    check(false);
    for (id *p = begin(); p < next; p++) {
        if (*p == POOL_BOUNDARY) {
            _objc_inform("[%p]  ################  POOL %p", p, p);
        } else {
            _objc_inform("[%p]  %#16lx  %s",
                         p, (unsigned long)*p, object_getClassName(*p));
        }
    }
}
```

- 在main.m 中添加如下代码
```
extern void _objc_autoreleasePoolPrint(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        //循环创建对象，并加入自动释放池
        for (int i = 0; i < 5; i++) {
             NSObject *objc = [[NSObject alloc] autorelease];
            NSLog(@"%p", objc);
        }
        
        //调用
        _objc_autoreleasePoolPrint();
    }
    return 0;
}
```
运行项目，打印结果如下：
```
2024-03-09 20:15:22.064402+0800 DMAutoreleasePoolDemo[38628:1055312] 0x105254ab0
2024-03-09 20:15:22.064568+0800 DMAutoreleasePoolDemo[38628:1055312] 0x105254710
2024-03-09 20:15:22.064587+0800 DMAutoreleasePoolDemo[38628:1055312] 0x105242740
2024-03-09 20:15:22.064602+0800 DMAutoreleasePoolDemo[38628:1055312] 0x105240d20
2024-03-09 20:15:22.064616+0800 DMAutoreleasePoolDemo[38628:1055312] 0x105240ca0
objc[38628]: ##############
objc[38628]: AUTORELEASE POOLS for thread 0x10008c580
objc[38628]: 6 releases pending.
objc[38628]: [0x10600a000]  ................  PAGE  (hot) (cold)
objc[38628]: [0x10600a038]  ################  POOL 0x10600a038
objc[38628]: [0x10600a040]       0x105254ab0  NSObject
objc[38628]: [0x10600a048]       0x105254710  NSObject
objc[38628]: [0x10600a050]       0x105242740  NSObject
objc[38628]: [0x10600a058]       0x105240d20  NSObject
objc[38628]: [0x10600a060]       0x105240ca0  NSObject
objc[38628]: ##############
Program ended with exit code: 0
```
从打印结果我们看到有6个对象，但是我们压栈的对象是5个，另一个其实是前面说到的哨兵对象（边界），目的是为了防止越界。另外，从地址的打印，我们也看到了哨兵对象与首地址相差了0x38（十进制56 位）刚好就是 AutoreleasePoolPage 所占的内存大小。

- 将上述for循环改为505次，再次运行项目，查看打印结果
```
objc[38809]: ##############
objc[38809]: AUTORELEASE POOLS for thread 0x10008c580
objc[38809]: 506 releases pending.
objc[38809]: [0x10580b000]  ................  PAGE (full)  (cold)
objc[38809]: [0x10580b038]  ################  POOL 0x10580b038
objc[38809]: [0x10580b040]       0x10070e770  NSObject
objc[38809]: [0x10580b048]       0x101067ca0  NSObject
objc[38809]: [0x10580b050]       0x101068200  NSObject
objc[38809]: [0x10580b058]       0x101060170  NSObject
objc[38809]: [0x10580b060]       0x101060a10  NSObject
objc[38809]: [0x10580b068]       0x101062b80  NSObject
objc[38809]: [0x10580b070]       0x1010606c0  NSObject
objc[38809]: [0x10580b078]       0x10105fee0  NSObject
objc[38809]: [0x10580b080]       0x10071a3d0  NSObject
......................................................
......................................................
......................................................
objc[38809]: [0x10580bfc8]       0x10122b8a0  NSObject
objc[38809]: [0x10580bfd0]       0x10122b8b0  NSObject
objc[38809]: [0x10580bfd8]       0x10122b8c0  NSObject
objc[38809]: [0x10580bfe0]       0x10122b8d0  NSObject
objc[38809]: [0x10580bfe8]       0x10122b8e0  NSObject
objc[38809]: [0x10580bff0]       0x10122b8f0  NSObject
objc[38809]: [0x10580bff8]       0x10071aec0  NSObject
objc[38809]: [0x105814000]  ................  PAGE  (hot) 
objc[38809]: [0x105814038]       0x10071aed0  NSObject
objc[38809]: ##############
```
从打印结果可以看到，第一页已经存满了，存储了504个需要释放的对象，第二页存储了一个对象。
- 将上述for循环改为1010次，再次巡行项目，查看打印结果
```
objc[39028]: ##############
objc[39028]: AUTORELEASE POOLS for thread 0x10008c580
objc[39028]: 1011 releases pending.
objc[39028]: [0x10080c000]  ................  PAGE (full)  (cold)
objc[39028]: [0x10080c038]  ################  POOL 0x10080c038
objc[39028]: [0x10080c040]       0x10100faa0  NSObject
objc[39028]: [0x10080c048]       0x101109390  NSObject
objc[39028]: [0x10080c050]       0x101108c20  NSObject
......................................................
......................................................
......................................................
objc[39028]: [0x10080cfd0]       0x10110a0a0  NSObject
objc[39028]: [0x10080cfd8]       0x101404820  NSObject
objc[39028]: [0x10080cfe0]       0x10110a0b0  NSObject
objc[39028]: [0x10080cfe8]       0x1014047c0  NSObject
objc[39028]: [0x10080cff0]       0x10110a0c0  NSObject
objc[39028]: [0x10080cff8]       0x10102c030  NSObject
objc[39028]: [0x10080f000]  ................  PAGE (full)  
objc[39028]: [0x10080f038]       0x10110a0d0  NSObject
objc[39028]: [0x10080f040]       0x10110a0e0  NSObject
objc[39028]: [0x10080f048]       0x10102c040  NSObject
objc[39028]: [0x10080f050]       0x10110a0f0  NSObject
objc[39028]: [0x10080f058]       0x1014047d0  NSObject
......................................................
......................................................
......................................................
objc[39028]: [0x10080ffd8]       0x101027800  NSObject
objc[39028]: [0x10080ffe0]       0x101405310  NSObject
objc[39028]: [0x10080ffe8]       0x101405320  NSObject
objc[39028]: [0x10080fff0]       0x10110ac50  NSObject
objc[39028]: [0x10080fff8]       0x101027810  NSObject
objc[39028]: [0x100811000]  ................  PAGE  (hot) 
objc[39028]: [0x100811038]       0x101027820  NSObject
objc[39028]: ##############
Program ended with exit code: 0
```
通过运行发现，第一页存储504个，第二页存储505个，第三页存储1个
> 自动释放池第一页存放1个哨兵对象加504个需要释放的对象，当第一页压栈满了，就会开辟新的一页，从第二页开始可以存放最多505个对象（一页的大小为505 * 8 = 4040字节）

同样这个结论可以通过 AutoreleasePoolPage 中 SIZE 来验证，定义中 PAGE_MAX_SIZE 大小为 4096字节，再起构造函数中对象的压栈位置 begin() 是从 首地址 +56 字节开始的，所以一个page中实际可以存储 4096 - 56 = 4040 字节，转换成对象 4040 / 8 = 505 个，因为第一页有哨兵对象，最多存储504个

![多个page布局]()

## 小结
整个自动释放池 AutoreleasePool 的事项以及 autorelease 方法都已经分析完了，我们再来回顾一下文章中的主要内容：
- 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的
- 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
- 调用 AutoreleasePoolPage::pop 方法会向栈中的对象发送 release 消息
