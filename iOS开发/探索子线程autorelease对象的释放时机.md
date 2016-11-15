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

##参考资料
1.[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
2.[autorelease使用注意事项](http://bbs.9ria.com/thread-223911-1-1.html)
3.[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
4.[Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/#jtss-tsina)
5.[runtime源代码](http://opensource.apple.com/tarballs/objc4/)
6.[libpthread](http://opensource.apple.com/tarballs/libpthread/)
