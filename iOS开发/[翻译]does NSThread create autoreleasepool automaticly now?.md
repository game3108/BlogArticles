## 前言
很不错的一个stackoverflow的问题，来自于[does NSThread create autoreleasepool automaticly now?](http://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automaticly-now)。

之前写过一篇[探索子线程autorelease对象的释放时机](http://www.jianshu.com/p/03f0c41410d9)，但感觉没有很好的深入找到问题根源，而这个答案就锁定了。
这里翻译一下，当作学习。

## 问题
I have test code like this
```
- (void)viewDidLoad
{
    [super viewDidLoad];
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(test) object:nil];
    [thread start];
}

-(void)test
{
    MyClass *my = [[[MyClass alloc] init] autorelease];
    NSLog(@"%@",[my description]);
}
```
I did not create any autoreleasepool for my own thread,but when the thread exit, object "my" just dealloc.why?

even though I change my test code as below

```
- (void)viewDidLoad
{
    [super viewDidLoad];

    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(test) object:nil];
    [thread start];
} 

-(void)test
{
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    MyClass *my = [[[MyClass alloc] init] autorelease];
    NSLog(@"%@",[my description]);
}
```
I create my own autoreleasepool but not drain it when the thread exit. object "my" can still dealloc anyway. why!!!!!!!??????

btw, I use Xcode5 and not using ARC

anyone can help me to figure out what happening now?

## 回答原文

It's not documented, but the answer appears to be **Yes**, on OS X 10.9+ and iOS 7+.
The Objective-C runtime is [open-source](http://www.opensource.apple.com/source/objc4/) so you can read the source to see what's going on. The latest version of the runtime (646, which shipped with OS X 10.10 and iOS 8) does indeed add a pool if you perform an autorelease
 without a pool on the current thread. In [NSObject.mm](http://www.opensource.apple.com/source/objc4/objc4-646/runtime/NSObject.mm):
```
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // No pool in place.
    assert(!hotPage());

    if (obj != POOL_SENTINEL  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place, 
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug", 
                     (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    // Push an autorelease pool boundary if it wasn't already requested.
    if (obj != POOL_SENTINEL) {
        page->add(POOL_SENTINEL);
    }

    // Push the requested object.
    return page->add(obj);
}
```
This function is called when you push the first pool (in which case the thing pushed is POOL_SENTINEL), or you autorelease with no pool. When the first pool is pushed, it sets up the autorelease stack. But as you see from the code, as long as the DebugMissingPools environmental variable is not set (it's not set by default), when autorelease is done with no pool, it also sets up the autorelease stack, and then pushes a pool (pushes a POOL_SENTINEL).

Similarly, (it's a little hard to follow without looking at the other code, but this is the relevant part) when the thread is destroyed (and the Thread-Local Storage is destroyed), it releases everything in the autorelease stack (that's what the pop(0); does) so it doesn't rely on the user to pop the last pool:
```
static void tls_dealloc(void *p) 
{
    // reinstate TLS value while we work
    setHotPage((AutoreleasePoolPage *)p);
    pop(0);
    setHotPage(nil);
}
```
The previous version of the runtime (551.1, which came with OS X 10.9 and iOS 7), also did this, as you can see from its [NSObject.mm](http://www.opensource.apple.com/source/objc4/objc4-551.1/runtime/NSObject.mm):
```
static __attribute__((noinline))
id *autoreleaseSlow(id obj)
{
    AutoreleasePoolPage *page;
    page = hotPage();

    // The code below assumes some cases are handled by autoreleaseFast()
    assert(!page || page->full());

    if (!page) {
        // No pool. Silently push one.
        assert(obj != POOL_SENTINEL);

        if (DebugMissingPools) {
            _objc_inform("MISSING POOLS: Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }

        push();
        page = hotPage();
    }

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
But the version before that (532.2, which came with OS X 10.8 and iOS 6), [does not](http://www.opensource.apple.com/source/objc4/objc4-532.2/runtime/NSObject.mm):
```
static __attribute__((noinline))
id *autoreleaseSlow(id obj)
{
    AutoreleasePoolPage *page;
    page = hotPage();

    // The code below assumes some cases are handled by autoreleaseFast()
    assert(!page || page->full());

    if (!page) {
        assert(obj != POOL_SENTINEL);
        _objc_inform("Object %p of class %s autoreleased "
                     "with no pool in place - just leaking - "
                     "break on objc_autoreleaseNoPool() to debug", 
                     obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return NULL;
    }

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```

Note that the above works for any pthreads, not just NSThreads.

So basically, if you are running on OS X 10.9+ or iOS 7+, autoreleasing on a thread without a pool should not lead to a leak. This is not documented and is an internal implementation detail, so be careful relying on this as Apple could change it in a future OS. However, I don't see any reason why they would remove this feature as it is simple and only has benefits and no downsides, unless they completely re-write the way autorelease pools work or something.

## 答案翻译
这没有被文档化，但这个答案显然是：是的。在OS X 10.9+和iOS 7+上。

Objective-c的runtime对你来说是开源的，所以你可以读源代码看如何运行。最新版本的runtime(646，用在OS X 10.10和IOS 8)确实添加了一个pool如果你仔没有pool的情况下运行autorelease。

```
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // No pool in place.
    assert(!hotPage());

    if (obj != POOL_SENTINEL  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place, 
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug", 
                     (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    // Push an autorelease pool boundary if it wasn't already requested.
    if (obj != POOL_SENTINEL) {
        page->add(POOL_SENTINEL);
    }

    // Push the requested object.
    return page->add(obj);
}
```
这个函数的调用时在你第一次压入pool（事实上push的是``POOL_SENTIAL``），或者你没有pool下autorelease。当第一个pool被压入，它建立autorelease的栈。但当你从code来看，因为``DebugMissingPools``的环境变量没有设置（不会被设为默认）。当autolrease没有pool下结束了，它同样会建立autorelease栈，然后压入一个pool(压入一个``POOL_SENTINEL``)

同样的，（如果没有看其他代码，有点难理解，但这个是相关部分）。当thread被销毁（同样线程本地的存储也被销毁），它释放了每一个在autorelease栈中的对象（使用``pop(0);``）因此它不建立在用户去弹出最后一个pool。

```
static void tls_dealloc(void *p) 
{
    // reinstate TLS value while we work
    setHotPage((AutoreleasePoolPage *)p);
    pop(0);
    setHotPage(nil);
}
```
这个当前版本的runtime(551.1，来自于OS X 10.9与iOS 7)也同样做了这个，所以你可以看``NSObject.mm``
```
static __attribute__((noinline))
id *autoreleaseSlow(id obj)
{
    AutoreleasePoolPage *page;
    page = hotPage();

    // The code below assumes some cases are handled by autoreleaseFast()
    assert(!page || page->full());

    if (!page) {
        // No pool. Silently push one.
        assert(obj != POOL_SENTINEL);

        if (DebugMissingPools) {
            _objc_inform("MISSING POOLS: Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }

        push();
        page = hotPage();
    }

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
但之前的版本(532.2，来自于OS X 10.8与 iOS 6)没有做。
```
static __attribute__((noinline))
id *autoreleaseSlow(id obj)
{
    AutoreleasePoolPage *page;
    page = hotPage();

    // The code below assumes some cases are handled by autoreleaseFast()
    assert(!page || page->full());

    if (!page) {
        assert(obj != POOL_SENTINEL);
        _objc_inform("Object %p of class %s autoreleased "
                     "with no pool in place - just leaking - "
                     "break on objc_autoreleaseNoPool() to debug", 
                     obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return NULL;
    }

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
注意上面的代码为任何``pthread``执行，不仅仅是``NSThread``。

基本上，如果你运行在OS X 10.9+或者iOS 7+，不用pool去autorelease在一个线程上不回引起内存泄漏。这个没有被文档记录并且是一个内部的实现细节，因此小心依赖这个因为Apple可能会在之后的系统修改它。然而，我看不到任何他们会移除这个功能的理由，因为这个简单，并且只有益处而没有坏处，除非他们重写autorelease pool的工作或者其他内容。

## 来源
[本文CSDN地址](http://blog.csdn.net/game3108/article/details/55051434)
[does NSThread create autoreleasepool automaticly now?](http://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automaticly-now)

