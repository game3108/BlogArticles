##前言
本文csdn地址：http://blog.csdn.net/game3108/article/details/51147949
今天在给同事讲autorelease对象释放时机：

**1.手动添加的autoreleasepool,pool release或者drain的时候释放**
MRC下
```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init;
// xxxxx
[pool drain];
```

ARC下
```
@autoreleasepool {
}
```
**2.runloop循环释放**
runloop循环中创建新的autoreleasepool并在循环结束中释放

autorelease原理在[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)有详细的介绍
runloop原理可以参考[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)

但一个子线程，默认是没有runloop的（第一次获得runloop对象时候创建），如果子线程上的autorelease对象何时释放？

**猜测**
* 子线程执行完任务被释放

测试：一个网上很多很多的例子（自己也写了一遍）
```
//
//  ViewController.m
//  Test
//
//  Created by game3108 on 16/3/14.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import "ViewController.h"
#import "TestObject.h"

@interface ViewController (){
    __weak id _testObject;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    TestObject *testObject = [TestObject instanceWithNumber:10];
    _testObject = testObject;
    
    UIButton *test = [[UIButton alloc]initWithFrame:CGRectMake(100, 100, 100, 100)];
    test.backgroundColor = [UIColor blackColor];
    [self.view addSubview:test];
    [test addTarget:self action:@selector(testButton) forControlEvents:UIControlEventTouchUpInside];
    // Do any additional setup after loading the view, typically from a nib.
}

- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    NSLog(@"viewWillAppear : %@", _testObject);
}

- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
    NSLog(@"viewDidAppear : %@", _testObject);
}

- (void)testButton{
    NSLog(@"testButton : %@", _testObject);
}
```

```
//
//  TestObject.h
//  Test
//
//  Created by game3108 on 16/4/5.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface TestObject : NSObject

+ (instancetype) instanceWithNumber:(NSInteger)number;

@end
```






```
//
//  TestObject.m
//  Test
//
//  Created by game3108 on 16/4/5.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import "TestObject.h"

@implementation TestObject{
    NSInteger _number;
}

+ (instancetype) instanceWithNumber:(NSInteger)number{
    //不加__autoreleasing 会因为arc返回值优化而不是一个autorelease对象，可以去掉试试
    __autoreleasing TestObject *object = [[TestObject alloc]init];
    object->_number = number;
    return object;
}

- (NSString *)description
{
    return [NSString stringWithFormat:@"%d", self->_number];
}

-(void)dealloc{
    NSLog(@"Test Object dealloc");
}

@end
```
输出结果也和网上的结果差不多：
```
2016-04-05 17:10:02.718 Test[978:534893] viewWillAppear : 10
2016-04-05 17:10:59.730 Test[978:534893] Test Object dealloc
2016-04-05 17:10:59.738 Test[978:534893] viewDidAppear : (null)
2016-04-05 17:11:10.564 Test[978:534893] testButton : (null)
```
给dealloc加一个断点
![dealloc断点图](http://upload-images.jianshu.io/upload_images/1829891-ab68f094cca480dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**嗯？怎么只有5行。。我当时看这个dealloc堆栈信息内心是有点崩溃的，好像系统调用都被隐藏了。也不知道那里去设置把他完整显示出来，最后捣鼓了半天，也只找到这个：（如果有人知道如何设置，请告诉我一下）**

![捣鼓出来的dealloc图](http://upload-images.jianshu.io/upload_images/1829891-06d0d3b25d97409a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总算看到了，``AutoreleasePoolPage:pop(void*)``一应俱全，也验证了相关文章中的信息：
runloop循环中，呼叫了相应的observer，进行autoreleasepool的创建和释放。


测试一下子线程的autorelease对象释放：

```
//
//  ViewController.m
//  Test
//
//  Created by game3108 on 16/3/14.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import "ViewController.h"
#import "TestObject.h"

@interface ViewController (){
    __weak id _testObject;
    NSThread *_testThread;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    _testThread = [[NSThread alloc] initWithTarget:self
                                                   selector:@selector(testBuild)
                                                     object:nil];
    [_testThread start];
    
    UIButton *test = [[UIButton alloc]initWithFrame:CGRectMake(100, 100, 100, 100)];
    test.backgroundColor = [UIColor blackColor];
    [self.view addSubview:test];
    [test addTarget:self action:@selector(testButton) forControlEvents:UIControlEventTouchUpInside];
    // Do any additional setup after loading the view, typically from a nib.
    
    [self performSelector:@selector(testAlert) onThread:_testThread withObject:nil waitUntilDone:NO];
}

- (void)testBuild{
    TestObject *testObject = [TestObject instanceWithNumber:10];
    _testObject = testObject;
}

- (void)testAlert{
    NSLog(@"test alert");
    NSLog(@"test alert");
    NSLog(@"test alert");
}

- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    NSLog(@"viewWillAppear : %@", _testObject);
}

- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
    NSLog(@"viewDidAppear : %@", _testObject);
}

- (void)testButton{
    NSLog(@"testButton : %@", _testObject);
}

@end
```

代码完成，同样的dealloc断点看一下：


![dealloc截图](http://upload-images.jianshu.io/upload_images/1829891-e972be706b0730ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**可以看到，当``pthread_exit``退出时，触发了``_pthread_tsd_cleanup``，触发AutoreleasePoolPage的``tls_dealloc(void*)``，然后回收autorelease对象**

使用``lldb``的``watchpoint``命令，给``_testObject = testObject;``增加断电，并且在命令行中``watchpoint set v _testObject``看一下，结果也是和刚才的差不多：


![watchpoint截图](http://upload-images.jianshu.io/upload_images/1829891-12585b93a51cf801.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那这边基本可以下一个结论：
**每一个线程创建的时候就会有一个autorelease pool的创建，并且在线程退出的时候，清空整个autorelease pool。（ps:如果在子线程中设置一个循环，autorelease对象确实无法释放）**


###总结：
**每个线程创建的时候就会创建一个autorelease pool，并且在线程退出的时候，清空autorelease pool。所以子线程的autorelease对象，要么在子线程中设置runloop清楚**

-------------------------------------------------------------------------

我下载了runtime的``objc4-680.tar.gz``，和libthread的``libpthread-138.10.4.tar.gz``,试图去理解一下整个过程（原谅我的渣C水平，要买几本书好好学习一下了。）：

NSObject.mm中class AutoreleasePoolPage的代码
找``tls_dealloc``
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

static void tls_dealloc(void *p) 
    {
        // reinstate TLS value while we work
        setHotPage((AutoreleasePoolPage *)p);

        if (AutoreleasePoolPage *page = coldPage()) {
            if (!page->empty()) pop(page->begin());  // pop all of the pools
            if (DebugMissingPools || DebugPoolAllocation) {
                // pop() killed the pages already
            } else {
                page->kill();  // free all of the pages
            }
        }
        
        // clear TLS value so TLS destruction doesn't loop
        setHotPage(nil);
    }
```
简单来说就是清空所有page
不停网上找调用点，会找到

NSObject.mm:
```
static void init()
    {
        int r __unused = pthread_key_init_np(AutoreleasePoolPage::key, 
                                             AutoreleasePoolPage::tls_dealloc);
        assert(r == 0);
    }
```
就会调用到pthread的方法，简单来讲就是会构造一个
```
static struct {
 uintptr_t destructor;
} _pthread_keys[_INTERNAL_POSIX_THREAD_KEYS_END];
```

将destructor方法保存，并且在thread退出的相应的时机调用。因为C水平有限，也只能懂个大概，也就不详细展开了。

##2017.2.13更新
我从stackoverflow [does NSThread create autoreleasepool automaticly now?](http://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automaticly-now)看到了一个更不错的答案，翻译一下，写在这里。

这没有被存档，但这个答案显然是：是的。在OS X 10.9+余iOS 7+上。

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


##参考资料
1.[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
2.[autorelease使用注意事项](http://bbs.9ria.com/thread-223911-1-1.html)
3.[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
4.[Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/#jtss-tsina)
5.[runtime源代码](http://opensource.apple.com/tarballs/objc4/)
6.[libpthread](http://opensource.apple.com/tarballs/libpthread/)
7.[does NSThread create autoreleasepool automaticly now?](http://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automaticly-now)
