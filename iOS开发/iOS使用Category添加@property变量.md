本文csdn地址：http://blog.csdn.net/game3108/article/details/51147941

由于Objective-C的动态语言特性，可以动态的创建类，添加属性和方法。
```
Class demoClass = objc_allocateClassPair([NSObject class], "TestDemo", 0);  
class_addMethod(demoClass, @selector(intro), (IMP)&selfIntro, "v@:");  
class_addIvar(demoClass,"myVar", sizeof(id), 0, "@");   
objc_registerClassPair(demoClass); 
```
其中就有一个``class_addIvar``可以动态添加变量。但``class_addIvar``只能在动态创建类的时候增加变量，定义类是静态创建的类则无法添加。

而Category只能添加方法，使用 @property 也是只会生成setter和getter方法的声明。具体原因请参考[这篇文章](http://chun.tips/blog/2014/11/06/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(3)%5Bnil%5D-xiao-xi-he-category/)，这文章详细解释了Category的实现，为什么可以动态添加方法，和不能添加实例变量的原因，这里也不多介绍了。

那如果确实想给Category增加一个@property的变量，应该如何去实现？
比如如下头文件：
```
//
//  UIView+Associate.h
//  TouchPalDialer
//
//  Created by game3108 on 16/3/30.
//
//

#import <UIKit/UIKit.h>

@interface UIView (Associate)

@property (nonatomic, assign) NSInteger associateLength;

@end
```

首先，先要了解
**@property的本质其实是 ivar(实例变量)+getter+setter**
那既然如此，重写getter和setter方法，并且增加一个ivar的变量，就可以实现@property的功能

对于重写getter和setter方法，可以用如下方式


```
//
//  UIView+Associate.m
//  TouchPalDialer
//
//  Created by game3108 on 16/3/30.
//
//

#import "UIView+Associate.h"

@implementation UIView (Associate)

@dynamic associateLength;

- (NSInteger) associateLength{
    NSLog(@"associate get");
return 0;
}

- (void) setAssociateLength:(NSInteger)associateLength{
    NSLog(@"associate length : %d" , associateLength);
}

@end
```

但重写getter和setter方法还是不够，必须要创建一个实例变量，这样，就用到了OC runtime动态绑定变量的方法：


``OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);``

``OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);``

这样可以动态创建一个ivar的实例变量使用，如下：


```
//
//  UIView+Associate.m
//  TouchPalDialer
//
//  Created by game3108 on 16/3/30.
//
//

#import "UIView+Associate.h"
#import <objc/runtime.h>

@implementation UIView (Associate)

@dynamic associateLength;

static char associateLengthKey;

- (NSInteger) associateLength{
    return [(NSNumber *)objc_getAssociatedObject(self, &associateLengthKey) integerValue];
}

- (void) setAssociateLength:(NSInteger)associateLength{
    objc_setAssociatedObject(self, &associateLengthKey, @(associateLength), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```
测试代码

```
UIView *testView = [[UIView alloc]init];
NSLog(@"test number: %d", testView.associateLength);
testView.associateLength = 20;
NSLog(@"test number: %d", testView.associateLength);
```

输出结果
``
test number: 0
test number: 20``

###### 参考链接
1.[刨根问底Objective－C Runtime（3）－ 消息 和 Category](http://chun.tips/blog/2014/11/06/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(3)%5Bnil%5D-xiao-xi-he-category/)
2.[Runtime-动态创建类添加属性和方法](http://blog.csdn.net/fucheng56/article/details/24032409)
3.[stackoverflow:objective-c-add-property-in-runtime](http://stackoverflow.com/questions/17888877/objective-c-add-property-in-runtime)
