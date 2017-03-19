## 随手记几个最近碰到的小问题
tips：如果有错误，或者有更好的详细解答，请随时联系我进行修改。

#### 1.UITextField输入卡住，字符不向左移动
发现UITextField在输入满以后，光标和输入位置卡住不动，内容text还在增加，但不会往左移。查了内部的UIFieldEditorContentView一切正常，无法理解。
后来发现是UITextField的输入框高度小了，比如字体18，高度20的情况，把高度改为24就正常了。

#### 2.实现单例的方式
iOS常见的实现单例方式目前有三种
1.initialize
2.加锁
3.GCD dispatch_once
注意的是，要防止直接alloc(new)创建对象的方式，所以要重写一些方法，拿GCD来举例
```
#import "SingleInstance.h"

@interface SingleInstance ()<NSCopying,NSMutableCopying>

@end

//定义一个当前单例对象的一个实例，并赋值为nil
static SingleInstance *instance = nil;

@implementation SingleInstance
+ (instancetype)sharedInstance
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });

    return instance;
}

//alloc会触发，防止通过alloc创建一个不同的实例
+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    
    return instance;
}

- (id)copyWithZone:(nullable NSZone *)zone
{
    return self;
}

- (id)mutableCopyWithZone:(nullable NSZone *)zone
{
    return self;
}

//手动内存管理下写这些

- (instancetype)retain
{
    return self;
}

- (oneway void)release
{
    
}

- (instancetype)autorelease
{
    return self;
}

- (NSUInteger)retainCount
{
    return MAXFLOAT;
}

- (void)dealloc
{
}
```

#### 3.查询crash log的uuid
每一个可执行程序都有一个build UUID来唯一标识。Crash日志包含发生crash的这个应用（app）的 build UUID以及crash发生的时候，应用加载的所有库文件的[build UUID]。

查询crash log的uuid可以用：``grep "appName armv" *crash``
或者``grep --after-context=2 "Binary Images:" *crash``
可以得到类似如下的结果：
```
appName.crash-0x4000 - 0x9e7fff appName armv7 <8bdeaf1a0b233ac199728c2a0ebb4165> 
/var/mobile/Applications/A0F8AB29-35D1-4E6E-84E2-954DE7D21CA1/appName.crash.app/appName
```
（请注意这里的0x4000，是模块的加载地址，后面用atos的时候会用到）

查询app的uuid可以使用命令：``xcrun dwarfdump -–uuid <AppName.app/ExecutableName>``
比如：``xcrun dwarfdump --uuid appName.app/appName``
结果如下：
```
UUID: 8BDEAF1A-0B23-3AC1-9972-8C2A0EBB4165 (armv7) appName.app/appName
UUID: 5EA16BAC-BB52-3519-B218-342455A52E11 (armv7s) appName.app/appName
```
这个app有2个UUID，表明它是一个fat binnary。
它能利用最新硬件的特性，又能兼容老版本的设备。
对比上面crash文件和app文件的UUID，发现它们是匹配的
```8BDEAF1A-0B23-3AC1-9972-8C2A0EBB4165```

**至于如何查crash更多的内容，可以参考参考资料。**

#### 4.NSTimer的持有对象问题
NSTimer使用容易有两大问题
1. 持有target（target再持有NSTimer就是引用循环，就算不持有，也释放不了)
2. RunloopMode的设置

这里主要说说怎么解决被持有的问题。
目前解决NSTimer的持有有两种方式
1. block方法
2. proxy代理方法

iOS10特别新增了block的支持方式，去解决被持有的问题。
其实这两种方式本身都是一样，就是修改NSTimer持有的target对象，让需要释放的对象不被持有，就可以在dealloc中去invalidate timer。
具体实现可以参考参考资料，这里不详细说了。




## 参考资料
[CSDN地址](http://blog.csdn.net/game3108/article/details/62885726)
1.[iOS崩溃crash大解析](http://www.qidiandasheng.com/2016/04/10/crash-xuebeng/)
2.[分析iOS Crash文件：符号化iOS Crash文件的3种方法](http://wufawei.com/2014/03/symbolicating-ios-crash-logs/)
3.[NSTimer和实现弱引用的timer的方式](http://blog.csdn.net/yohunl/article/details/50614903)
