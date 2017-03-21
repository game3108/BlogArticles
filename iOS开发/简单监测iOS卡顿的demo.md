## 前言
本文的demo代码也会更新到[github](https://github.com/game3108/RunloopMonitorDemo)上。

做这个demo思路来源于微信team的：[微信iOS卡顿监控系统](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207890859&idx=1&sn=e98dd604cdb854e7a5808d2072c29162&scene=4#wechat_redirect)。
主要思路:通过监测Runloop的kCFRunLoopAfterWaiting，用一个子线程去检查，一次循环是否时间太长。
其中主要涉及到了runloop的原理。关于整个原理：[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)讲解的比较仔细。
以下就是runloop大概的运行方式：
```{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);

/// 5. GCD处理main block
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

其中UI主要集中在``__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);``
和``__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);``之前。
获取``kCFRunLoopBeforeSources``到``kCFRunLoopBeforeWaiting``再到``kCFRunLoopAfterWaiting``的状态就可以知道是否有卡顿的情况。

## NSTimer的实现
具体代码如下：
```
//
//  MonitorController.h
//  RunloopMonitorDemo
//
//  Created by game3108 on 16/4/13.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface MonitorController : NSObject
+ (instancetype) sharedInstance;
- (void) startMonitor;
- (void) endMonitor;
- (void) printLogTrace;
@end
```


```
//
//  MonitorController.m
//  RunloopMonitorDemo
//
//  Created by game3108 on 16/4/13.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import "MonitorController.h"
#include <libkern/OSAtomic.h>
#include <execinfo.h>

@interface MonitorController(){
    CFRunLoopObserverRef _observer;
    double _lastRecordTime;
    NSMutableArray *_backtrace;
}

@end

@implementation MonitorController

static double _waitStartTime;

+ (instancetype) sharedInstance{
    static dispatch_once_t once;
    static id sharedInstance;
    dispatch_once(&once, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void) startMonitor{
    [self addMainThreadObserver];
    [self addSecondaryThreadAndObserver];
}

- (void) endMonitor{
    if (!_observer) {
        return;
    }
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
    CFRelease(_observer);
    _observer = NULL;
}

#pragma mark printLogTrace
- (void)printLogTrace{
    NSLog(@"====================堆栈\n %@ \n",_backtrace);
}

#pragma mark addMainThreadObserver
- (void) addMainThreadObserver {
    dispatch_async(dispatch_get_main_queue(), ^{
        //建立自动释放池
        @autoreleasepool {
            //获得当前thread的Run loop
            NSRunLoop *myRunLoop = [NSRunLoop currentRunLoop];
            
            //设置Run loop observer的运行环境
            CFRunLoopObserverContext context = {0, (__bridge void *)(self), NULL, NULL, NULL};
            
            //创建Run loop observer对象
            //第一个参数用于分配observer对象的内存
            //第二个参数用以设置observer所要关注的事件，详见回调函数myRunLoopObserver中注释
            //第三个参数用于标识该observer是在第一次进入run loop时执行还是每次进入run loop处理时均执行
            //第四个参数用于设置该observer的优先级
            //第五个参数用于设置该observer的回调函数
            //第六个参数用于设置该observer的运行环境
            _observer = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
            
            if (_observer) {
                //将Cocoa的NSRunLoop类型转换成Core Foundation的CFRunLoopRef类型
                CFRunLoopRef cfRunLoop = [myRunLoop getCFRunLoop];
                //将新建的observer加入到当前thread的run loop
                CFRunLoopAddObserver(cfRunLoop, _observer, kCFRunLoopDefaultMode);
            }
        }
    });
}

void myRunLoopObserver(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    switch (activity) {
            //The entrance of the run loop, before entering the event processing loop.
            //This activity occurs once for each call to CFRunLoopRun and CFRunLoopRunInMode
        case kCFRunLoopEntry:
            NSLog(@"run loop entry");
            break;
            //Inside the event processing loop before any timers are processed
        case kCFRunLoopBeforeTimers:
            NSLog(@"run loop before timers");
            break;
            //Inside the event processing loop before any sources are processed
        case kCFRunLoopBeforeSources:
            NSLog(@"run loop before sources");
            break;
            //Inside the event processing loop before the run loop sleeps, waiting for a source or timer to fire.
            //This activity does not occur if CFRunLoopRunInMode is called with a timeout of 0 seconds.
            //It also does not occur in a particular iteration of the event processing loop if a version 0 source fires
        case kCFRunLoopBeforeWaiting:{
            _waitStartTime = 0;
            NSLog(@"run loop before waiting");
            break;
        }
            //Inside the event processing loop after the run loop wakes up, but before processing the event that woke it up.
            //This activity occurs only if the run loop did in fact go to sleep during the current loop
        case kCFRunLoopAfterWaiting:{
            _waitStartTime = [[NSDate date] timeIntervalSince1970];
            NSLog(@"run loop after waiting");
            break;
        }
            //The exit of the run loop, after exiting the event processing loop.
            //This activity occurs once for each call to CFRunLoopRun and CFRunLoopRunInMode
        case kCFRunLoopExit:
            NSLog(@"run loop exit");
            break;
            /*
             A combination of all the preceding stages
             case kCFRunLoopAllActivities:
             break;
             */
        default:
            break;
    }
}

#pragma mark addSecondaryThreadAndObserver
- (void) addSecondaryThreadAndObserver{
    NSThread *thread = [self secondaryThread];
    [self performSelector:@selector(addSecondaryTimer) onThread:thread withObject:nil waitUntilDone:YES];
}

- (NSThread *)secondaryThread {
    static NSThread *_secondaryThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _secondaryThread =
        [[NSThread alloc] initWithTarget:self
                                selector:@selector(networkRequestThreadEntryPoint:)
                                  object:nil];
        [_secondaryThread start];
    });
    return _secondaryThread;
}

- (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"monitorControllerThread"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
        [runLoop run];
    }
}

- (void) addSecondaryTimer{
    NSTimer *myTimer = [NSTimer timerWithTimeInterval:0.5 target:self selector:@selector(timerFired:) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:myTimer forMode:NSDefaultRunLoopMode];
}

- (void)timerFired:(NSTimer *)timer{
    if ( _waitStartTime < 1 ){
        return;
    }
    double currentTime = [[NSDate date] timeIntervalSince1970];
    double timeDiff = currentTime - _waitStartTime;
    if (timeDiff > 2.0){
        if (_lastRecordTime - _waitStartTime < 0.001 && _lastRecordTime != 0){
            NSLog(@"last time no :%f %f",timeDiff, _waitStartTime);
            return;
        }
        [self logStack];
        _lastRecordTime = _waitStartTime;
    }
}

- (void)logStack{
    void* callstack[128];
    int frames = backtrace(callstack, 128);
    char **strs = backtrace_symbols(callstack, frames);
    int i;
    _backtrace = [NSMutableArray arrayWithCapacity:frames];
    for ( i = 0 ; i < frames ; i++ ){
        [_backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
    }
    free(strs);
}

@end
```

主要内容是首先在主线程注册了runloop observer的回调``myRunLoopObserver``
每次小循环都会记录一下``kCFRunLoopAfterWaiting``的时间``_waitStartTime``，并且在``kCFRunLoopBeforeWaiting``制空。

另外开了一个子线程并开启他的runloop（模仿了AFNetworking的方式），并加上一个timer每隔1秒去进行监测。

如果当前时长与``_waitStartTime``差距大于2秒，则认为有卡顿情况，并记录了当前堆栈信息。

PS:整个demo写的比较简单，最后获取堆栈也仅获取了当前线程的堆栈信息(``[NSThread callStackSymbols]``有同样效果)，也在寻找获取所有线程堆栈的方法，欢迎指点一下。

------------------------------------------------------

#### 更新：
了解到 plcrashreporter ([github地址](https://github.com/plausiblelabs/plcrashreporter)) 可以做到获取所有线程堆栈。

------------------------------------------------------

#### 更新2:
这篇文章也介绍了监测卡顿的方法：[检测iOS的APP性能的一些方法](http://www.starming.com/index.php?v=index&view=91)
通过Dispatch Semaphore保证同步这里记录一下。

写一个Semaphore版本的代码，也放在[github](https://github.com/game3108/RunloopMonitorDemo)上：
```
//
//  SeMonitorController.h
//  RunloopMonitorDemo
//
//  Created by game3108 on 16/4/14.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface SeMonitorController : NSObject
+ (instancetype) sharedInstance;
- (void) startMonitor;
- (void) endMonitor;
- (void) printLogTrace;
@end
```

```
//
//  SeMonitorController.m
//  RunloopMonitorDemo
//
//  Created by game3108 on 16/4/14.
//  Copyright © 2016年 game3108. All rights reserved.
//

#import "SeMonitorController.h"
#import <libkern/OSAtomic.h>
#import <execinfo.h>

@interface SeMonitorController(){
    CFRunLoopObserverRef _observer;
    dispatch_semaphore_t _semaphore;
    CFRunLoopActivity _activity;
    NSInteger _countTime;
    NSMutableArray *_backtrace;
}

@end

@implementation SeMonitorController

+ (instancetype) sharedInstance{
    static dispatch_once_t once;
    static id sharedInstance;
    dispatch_once(&once, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void) startMonitor{
    [self registerObserver];
}

- (void) endMonitor{
    if (!_observer) {
        return;
    }
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
    CFRelease(_observer);
    _observer = NULL;
}

- (void) printLogTrace{
    NSLog(@"====================堆栈\n %@ \n",_backtrace);
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    SeMonitorController *instrance = [SeMonitorController sharedInstance];
    instrance->_activity = activity;
    // 发送信号
    dispatch_semaphore_t semaphore = instrance->_semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)registerObserver
{
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    _observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                            kCFRunLoopAllActivities,
                                                            YES,
                                                            0,
                                                            &runLoopObserverCallBack,
                                                            &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
    
    // 创建信号
    _semaphore = dispatch_semaphore_create(0);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            // 假定连续5次超时50ms认为卡顿(当然也包含了单次超时250ms)
            long st = dispatch_semaphore_wait(_semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (_activity==kCFRunLoopBeforeSources || _activity==kCFRunLoopAfterWaiting)
                {
                    if (++_countTime < 5)
                        continue;
                    [self logStack];
                    NSLog(@"something lag");
                }
            }
            _countTime = 0;
        }
    });
}

- (void)logStack{
    void* callstack[128];
    int frames = backtrace(callstack, 128);
    char **strs = backtrace_symbols(callstack, frames);
    int i;
    _backtrace = [NSMutableArray arrayWithCapacity:frames];
    for ( i = 0 ; i < frames ; i++ ){
        [_backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
    }
    free(strs);
}

@end
```
用Dispatch Semaphore简化了代码复杂度，更加简洁。


## 参考资料
[本文csdn地址](http://blog.csdn.net/game3108/article/details/51147946)
1.[微信iOS卡顿监控系统](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207890859&idx=1&sn=e98dd604cdb854e7a5808d2072c29162&scene=4#wechat_redirect)
2.[ [iphone——使用run loop对象](http://blog.csdn.net/lingedeng/article/details/6870692)](http://blog.csdn.net/lingedeng/article/details/6870692)
3.[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
4.[检测iOS的APP性能的一些方法](http://www.starming.com/index.php?v=index&view=91)
5.[iOS实时卡顿监控](http://www.tanhao.me/code/151113.html/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
