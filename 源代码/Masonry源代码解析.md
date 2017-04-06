## 前言
CSDN地址：http://blog.csdn.net/game3108/article/details/52473932
本文的中文注释代码demo更新在我的[github](https://github.com/game3108/MasonryDemo)上。

AutoLayout是Apple在iOS6中新增的UI布局适配的方法，用来替代iOS6之前的AutoResizeing。AutoLayout对应的代码约束就是NSLayoutConstraint。NSLayoutConstraint的API虽说时分简单，但约束的代码量较大，所以出现了很多对NSLayoutConstraint的封装，今天讲的就是其中最为有名的Masonry框架。

[Masonry](https://github.com/SnapKit/Masonry)框架简化了约束NSLayoutConstraint的写法，在各种APP中都有很多的使用。Masonry是基于Objective-C语言的框架，Swift项目可以参考[Snapkit](https://github.com/SnapKit/SnapKit)框架。

本文会基于Masonry v1.0.1版本，同时借鉴了网上很多解析文章，对源代码进行解析，进行学习。

***
16.9.14更新：
MASViewConstraint的``equalToWithRelation``添加array缺少``//viewConstraint.layoutRelation = relation;``的pull request已经被接受，该问题fixed。

## 约束

NSLayoutConstraint约束是基于以下公式：

>item1.attribute1 = multiplier × item2.attribute2 + constant

比如button1的左侧距离button2有10的约束会这么写：
```
button1.left = button2.right + 10;
```

事实上添加NSLayoutConstraint约束有三种方式：
* 1.storyboard/xib添加
* 2.VFL语言添加
* 3.NSLayoutConstraint纯代码添加

### 1.storyboard/xib添加
storyboard/xib添加NSLayoutConstraint的方式主要是右下角的约束设置：

![xib右下角](http://upload-images.jianshu.io/upload_images/1829891-8199b64193978449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分别是：
* **Align**
* **Pin**
* **Resolve Auto Layout Issues**

相应的图这里就不再贴了，有兴趣的可以自己去试一下

### 2.VFL语言添加
VFL（Visual Format Language）是苹果公司为了简化autolayout的编码而推出的抽象语言。
VFL调用以下方法：
```
+ (NSArray<__kindof NSLayoutConstraint *> *)constraintsWithVisualFormat:(NSString *)format 
options:(NSLayoutFormatOptions)opts 
metrics:(nullable NSDictionary<NSString *,id> *)metrics 
views:(NSDictionary<NSString *, id> *)views;
```
其中的``format``就是vfl语句。
vfl的语句也较为复杂，这里不详细介绍了，具体可以参考苹果文档[Visual Format Language](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html)。

### 3.NSLayoutConstraint纯代码添加
我们举一个例子：
view的上部距离距离superview有10的距离：
```
NSLayoutConstraint *constraint = [NSLayoutConstraint constraintWithItem:view                        //view
attribute:NSLayoutAttributeTop        //view.top
relatedBy:NSLayoutRelationEqual       //view.top =
toItem:superView                   //view.top = superView
attribute:NSLayoutAttributeTop        //view.top = superView.top
multiplier:1.0                         //view.top = 1.0 * superView.top
constant:10.0];                      //view.top = 1.0 * superView.top + 10
[view addConstraint:constraint];
```
可以看到约束代码算是列出了一个约束公式，也达到了约束的目的。
但这些代码，其实只写了一个top的约束，如果有其它约束代码，需要同样写类似的代码出来，所以直接用NSLayoutConstraint纯代码添加还是比较麻烦的一件事。

### 4.约束的限制
（1）对于两个同层级 view 之间的约束关系，添加到它们的父 view 上
（2）对于两个不同层级 view 之间的约束关系，添加到他们最近的共同父 view 上
（3）对于有层次关系的两个 view 之间的约束关系，添加到层次较高的父 view 上
（4）对于比如长宽之类的，只作用在该 view 自己身上的话，添加到该 view 自己上

## Masonry的使用
###1.Masonry的例子
Masonry的代码封装了NSLayoutConstraint纯代码，简洁了许多，和NSLayoutConstraint纯代码举同一个例子：
view的上部距离距离superview有10的距离
```
[view mas_makeConstraints:^(MASConstraintMaker *make){
make.top.equalTo(10);  //默认是父view 三者等价  view.top = 1.0 * superView.top + 10

make.top.mas_equalTo(superView.mas_top).with.multipliedBy(1.0).mas_offset(10);

make.top.equalTo(superView.top).offset(10);
}];
```
相对于NSLayoutConstraint纯代码的添加约束，Masonry使用了block外加链式语法，使得调用简洁和方便了许多。

## Masonry源代码
### 1.整体结构
Masonry的目录结构如下：

![Masonry](http://upload-images.jianshu.io/upload_images/1829891-d12dadfdce94e2ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

文件比较多，借用[iOS开发之Masonry框架源码深度解析](http://www.cnblogs.com/ludashi/p/5591572.html)的类图：

![iOS开发之Masonry框架源码深度解析-Masonry类图](http://upload-images.jianshu.io/upload_images/1829891-2157823aedb4e4dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据类图，文件目录主要分为以下几类：

* 1.头文件和辅助文件
Masonry.h:头文件
MASUtilities.h:定义宏MASBoxValue，转换类型
UIView+MASShorthandAdditions.h:定义UIView调用的简化参数和方法
NSArray+MASShorthandAdditions.h：定义NSArray调用的简化参数和方法
NSLayoutConstraint+MASDebugAdditions.h/m:DEBUG相关信息转换

* 2.约束使用接口
UIView+MASAdditions.h/m:UIView的约束接口
UIViewController+MASAdditions.h/m:UIViewController的约束借口
NSArray+MASAdditions.h/m:遍历NSArray中的UIView的约束接口

* 3.约束建造者builder模式
MASConstraintMaker.h/m：约束构造使用的建造者builder

* 4.约束内部结构和实现
MASLayoutConstraint.h/m：NSLayoutConstraint多了一层key的封装
MASAttribute.h:NSLayoutAttribute的一层封装
MASConstraint.h/m:定义链式结构体的抽象父类
MASConstraint+Private.h:定义MASConstraint的接口和delegate代理
MASViewConstraint.h/m:继承MASConstraint，表示view的约束结构体
MASCompositeConstraint.h/m:继承MASConstraint，表示系列view组合的约束结构体

**看了源代码会发现，Masonry的代码流程简单来讲就是：提供给用户一个建造者MASConstraintMaker，让用户根据mansory提供的语法，添加约束结构体MASConstraint。最后Masonry解析约束结构体MASConstraint，将真正的约束关系NSLayoutConstraint添加到相应的view上。**

### 2.源代码探究
我们根据一个实际调用代码来讲Masonry的源代码。代码如下：
```
[view mas_makeConstraints:^(MASConstraintMaker *make){
make.top.mas_equalTo(superView.mas_top).with.mas_offset(10);
}];
```
这边其实主要分两块
* MASConstraintMaker的链式调用
* view的约束函数调用

在讲这两块前，首先讲一下底层的数据结构MASConstraint相关的结构

#### （1）MASConstraint相关结构
MASConstraint是定义约束的抽象类（虽然OC没有抽象类的说法）
定义大概如下:
```
@interface MASConstraint : NSObject
...
- (MASConstraint * (^)(CGFloat offset))offset;
...
- (MASConstraint *)left;
- (MASConstraint *)top;
...
- (void)install;
- (void)uninstall;
```
这边只选取了部分方法和参数，来看一下相应的实现

```
- (MASConstraint * (^)(CGFloat))offset {
return ^id(CGFloat offset){
self.offset = offset;
return self;
};
}


//让子类去重写该方法，父类抛出异常达到抽象类的效果
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute __unused)layoutAttribute {
MASMethodNotImplemented();
}

- (MASConstraint *)left {
return [self addConstraintWithLayoutAttribute:NSLayoutAttributeLeft];
}

- (void)install { MASMethodNotImplemented(); }

- (void)uninstall { MASMethodNotImplemented(); }
```
可以看到这边的实现达到了以下目的：
* 1.链式调用的实现
为了实现链式调用，并且可以用小括号"()"加入参数，而不是中括号"[]"，这边采用了返回同样类型的block的方式
* 2.让子类实现方法，父类作为抽象类存在
虽然oc没有抽象类的定义，但在MASConstraint定义了一系列方法，让子类进行重写，而父类则用宏``MASMethodNotImplemented()``抛出异常

而MASConstraint的子类则分别是MASViewConstraint与MASCompositeConstraint
##### MASViewConstraint
MASViewConstraint定义着一个单独的约束关系
MASViewConstraint.h中定义如下：
```
@interface MASViewConstraint : MASConstraint <NSCopying>
//代表第一个item和相应的attribute
@property (nonatomic, strong, readonly) MASViewAttribute *firstViewAttribute;
//代表第二个item和相应的attribute
@property (nonatomic, strong, readonly) MASViewAttribute *secondViewAttribute;
...
```
这里又多了一个类MASViewAttribute

```
@interface MASViewAttribute : NSObject
//调用view
@property (nonatomic, weak, readonly) MAS_VIEW *view;
//交互对象
@property (nonatomic, weak, readonly) id item;
//约束NSLayoutAttribute类型
@property (nonatomic, assign, readonly) NSLayoutAttribute layoutAttribute;
```
MASViewAttribute比较好理解，其实就是一个调用view，交互的item和NSLayoutAttribute的结构体
**而为什么在MASViewConstraint中会有两个MASViewAttribute？**
我们再看一下一个标准的约束调用代码：
```
NSLayoutConstraint *constraint = [NSLayoutConstraint constraintWithItem:view                        //view
attribute:NSLayoutAttributeTop        //view.top
relatedBy:NSLayoutRelationEqual       //view.top =
toItem:superView                   //view.top = superView
attribute:NSLayoutAttributeTop        //view.top = superView.top
multiplier:1.0                         //view.top = 1.0 * superView.top
constant:10.0];                      //view.top = 1.0 * superView.top + 10
```
**可以看到，一个约束必须要有两组<item,NSLayoutAttribute>的结构体，还需要NSLayoutRelation\multiplier\constant。而这两个MASViewAttribute就是对应两组两组<item,NSLayoutAttribute>的结构体。**

##### MASCompositeConstraint
MASCompositeConstraint代表一组MASViewConstraint约束
比如源代码中的``make.top``会转化为MASViewConstraint约束，而``make.top.bottom.left.right.xxxxx``这种写法会定义多种MASViewConstraint约束，而为了存储这种写法，则创建了MASCompositeConstraint。

MASCompositeConstraint.h定义并没有暴露太多细节，细节都在.m的匿名extension中：

```
@interface MASCompositeConstraint () <MASConstraintDelegate>
@property (nonatomic, strong) id mas_key;
//存储内部结构体，都是MASViewConstraint
@property (nonatomic, strong) NSMutableArray *childConstraints;
@end
```
childConstraints中存储的是每一条MASViewConstraint约束

#### （2）MASConstraintMaker调用
MASConstraintMaker匿名Extension中的定义
```
@interface MASConstraintMaker () <MASConstraintDelegate>
@property (nonatomic, weak) MAS_VIEW *view;
//存储MASConstraint
@property (nonatomic, strong) NSMutableArray *constraints;
@end
```
可以看到在有一个MASConstraintMaker中专门有一个``@property (nonatomic, strong) NSMutableArray *constraints;``去存储所有的MASConstraint对象。就是你有好几行的make.xxx都会存储在这里。

MASConstraintMaker.h中添加了许多的MASConstraint property
```
@interface MASConstraintMaker : NSObject
//property 重写了getter方法，使得每次链式调用相当于构造一个约束
@property (nonatomic, strong, readonly) MASConstraint *left;
@property (nonatomic, strong, readonly) MASConstraint *top;
...
```
并重写了它们的getter方法：
```
//通用增加约束的方法
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

//重写getter方法，调用.方法相当于构造约束
- (MASConstraint *)left {
return [self addConstraintWithLayoutAttribute:NSLayoutAttributeLeft];
}

- (MASConstraint *)top {
return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}
```
相当于我make调用相应的property，就是给给view添加了某个NSLayoutAttribute的结构体

```
//替换函数
- (void)constraint:(MASConstraint *)constraint shouldBeReplacedWithConstraint:(MASConstraint *)replacementConstraint {
NSUInteger index = [self.constraints indexOfObject:constraint];
NSAssert(index != NSNotFound, @"Could not find constraint %@", constraint);
[self.constraints replaceObjectAtIndex:index withObject:replacementConstraint];
}

//通过NSLayoutAttribute添加约束
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
//构造view的MASViewAttribute
MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
//通过MASViewAttribute构造第一个MASViewConstraint
MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
//如果存在contraint，则把constraint和newConstraint组合成MASCompositeConstraint
if ([constraint isKindOfClass:MASViewConstraint.class]) {
//replace with composite constraint
NSArray *children = @[constraint, newConstraint];
MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
compositeConstraint.delegate = self;
//替换原来的constraint成新的MASCompositeConstraint
[self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
return compositeConstraint;
}
//不存在则设置constraint到self.constraints
if (!constraint) {
newConstraint.delegate = self;
[self.constraints addObject:newConstraint];
}
return newConstraint;
}
```
举一个实际的例子：
```
make.top.mas_equalTo(superView.mas_top).with.mas_offset(10);
```

当调用``make.top``的时候就会创建一个只只有firstViewAttribute的MASViewConstraint对象，并且进入不存在约束constraint的代码部分
```
if (!constraint) {
newConstraint.delegate = self;
[self.constraints addObject:newConstraint];
}
```
设置delegate，并且将约束添加到self.constraints，同时返回的是刚刚创建的MASViewConstraint对象。

如果调用代码是``make.top.left``到left的时候其实是MASViewConstraint对象.left的调用，会走到我们刚才说的MASViewConstraint中重写的``- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute``方法，实现如下：

MASViewConstraint.m
```
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
NSAssert(!self.hasLayoutRelation, @"Attributes should be chained before defining the constraint relation");
return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}
```
这边就调用了MASViewConstraint的delegate方法，而MASViewConstraint的delegate对象则是刚才设置过的MASConstraintMaker对象，回到了刚才添加约束的方法。又新建了一个MASViewConstraint对象，并且因为constraint不是nil，进入
```
if ([constraint isKindOfClass:MASViewConstraint.class]) {
//replace with composite constraint
NSArray *children = @[constraint, newConstraint];
MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
compositeConstraint.delegate = self;
//替换原来的constraint成新的MASCompositeConstraint
[self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
return compositeConstraint;
}
```
返回的是一个MASCompositeConstraint对象。其中注意MASCompositeConstraint的初始化方法
```
- (id)initWithChildren:(NSArray *)children {
self = [super init];
if (!self) return nil;

_childConstraints = [children mutableCopy];
for (MASConstraint *constraint in _childConstraints) {
constraint.delegate = self;
}

return self;
}
```
会把里面的MASViewConstraint的delegate全部设置到MASCompositeConstraint对象身上。

如果调用的代码是``make.top.left.right``到right的时候，就是MASCompositeConstraint对象.right的调用，会走MASCompositeConstraint中的重写方法
```
- (MASConstraint *)constraint:(MASConstraint __unused *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
id<MASConstraintDelegate> strongDelegate = self.delegate;
MASConstraint *newConstraint = [strongDelegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
newConstraint.delegate = self;
[self.childConstraints addObject:newConstraint];
return newConstraint;
}
```
这边也会调用MASConstraintMaker的同一个方法，但这次``if ([constraint isKindOfClass:MASViewConstraint.class])``与``if (!constraint)``都不会进入，只会返回单纯的MASViewConstraint对象，然后设置它的delegate，并且将对象存入MASCompositeConstraint的childConstraints中。
之后再有更多的链式MASConstraint的组合，都是MASCompositeConstraint的调用，不停的加入childConstraints中而已。同理superView.mas_top的构成也是同样的方式。

直到调用到``mas_equalTo(xx)``
在MASConstraint中的实现是：
```
- (MASConstraint * (^)(id))equalTo {
return ^id(id attribute) {
return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
};
}
```
两个子类重写了``equalToWithRelation``该方法
MASViewConstraint
```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
return ^id(id attribute, NSLayoutRelation relation) {
//如果是array，则是代表是多个组合对象
if ([attribute isKindOfClass:NSArray.class]) {
//如果已经有relation，则报错
NSAssert(!self.hasLayoutRelation, @"Redefinition of constraint relation");
//组合约束存储
NSMutableArray *children = NSMutableArray.new;
//遍历每一个组合对象
for (id attr in attribute) {
//复制自己,这里有个逻辑，因为在copy的时候会设置一下layoutRelation，这边的setLayoutRelation
//方法会把hasLayoutRelation设置为YES，单layoutRelation是默认的NSLayoutRelationEqual
//这边其实比较明显有个bug，就是没有将relation在这里设置一下，所以所有的关系都会变成默认的
//我提交了一个pull request，等待修改
MASViewConstraint *viewConstraint = [self copy];
//viewConstraint.layoutRelation = relation;
//设置设置second attribute
viewConstraint.secondViewAttribute = attr;
[children addObject:viewConstraint];
}
//初始化MASCompositeConstraint
MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
compositeConstraint.delegate = self.delegate;
//替换原来的位置
[self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];
return compositeConstraint;
} else {
//判断是否已经有过relation,有的话则报错
NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
//存储relation
self.layoutRelation = relation;
//设置second attribute
self.secondViewAttribute = attribute;
return self;
}
};
}

```
tips:**这边的代码有array添加的时候，会有一个relation设置问题的bug，少了``//viewConstraint.layoutRelation = relation;``这句话。我已经提交了pull request，等待修复该问题。**

MASCompositeConstraint
```
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
return ^id(id attr, NSLayoutRelation relation) {
//遍历所有的MASViewConstraint并设置
for (MASConstraint *constraint in self.childConstraints.copy) {
constraint.equalToWithRelation(attr, relation);
}
return self;
};
}
```
可以看到，该方法主要是为了设置layoutRelation与secondViewAttribute。
最后则是调用
```
- (MASConstraint * (^)(CGFloat))offset {
return ^id(CGFloat offset){
self.offset = offset;
return self;
};
}
```
设置偏移。
至此，整个构建创建者MASConstraintMaker的调用到此为止。

#### （3）约束函数调用

例子代码中的调用函数如下:
UIView+MASAdditions.m
```
//创建约束
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
//去掉自动autoresizing转为约束的
self.translatesAutoresizingMaskIntoConstraints = NO;
//构建builder
MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
//运行builder
block(constraintMaker);
//附值约束返回
return [constraintMaker install];
}
```
先是去掉了AutoResizing的自动转换，然后创建一个MASConstraintMaker对象，调用block去构成builder建造者。最后是install方法进行设置。

```
- (NSArray *)install {
//如果需要删除原来的约束
if (self.removeExisting) {
//获得所有约束并删除
NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
for (MASConstraint *constraint in installedConstraints) {
[constraint uninstall];
}
}
NSArray *constraints = self.constraints.copy;
for (MASConstraint *constraint in constraints) {
//设置更新key
constraint.updateExisting = self.updateExisting;
[constraint install];
}
//去除所有缓存的约束结构体
[self.constraints removeAllObjects];
return constraints;
}
```
先判断是否删除所有的约束，然后再添加需要更新的约束。

先看一下uninstall方法
MASCompositeConstraint.m
```
- (void)uninstall {
for (MASConstraint *constraint in self.childConstraints) {
[constraint uninstall];
}
}
```
MASConstraint.m
```
//删除约束
- (void)uninstall {
//判断是否支持active去设置NSLayoutConstraint
if ([self supportsActiveProperty]) {
//设置active为no
self.layoutConstraint.active = NO;
//删除installed缓存
[self.firstViewAttribute.view.mas_installedConstraints removeObject:self];
return;
}

//删除此约束
[self.installedView removeConstraint:self.layoutConstraint];
self.layoutConstraint = nil;
self.installedView = nil;
//删除installed缓存
[self.firstViewAttribute.view.mas_installedConstraints removeObject:self];
}
```
其中NSLayoutConstraint的active相当于``addConstraint:``和``removeConstraint:``。

然后就是install方法
MASCompositeConstraint.m
```
- (void)install {
for (MASConstraint *constraint in self.childConstraints) {
constraint.updateExisting = self.updateExisting;
[constraint install];
}
}
```
MASViewConstraint.m
```
- (void)install {
//已经有约束
if (self.hasBeenInstalled) {
return;
}

//支持active且已经有了约束
if ([self supportsActiveProperty] && self.layoutConstraint) {
//激活约束
self.layoutConstraint.active = YES;
//添加约束缓存
[self.firstViewAttribute.view.mas_installedConstraints addObject:self];
return;
}

//获得item1,attribute1,item2,attribute2
MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;

// alignment attributes must have a secondViewAttribute
// therefore we assume that is refering to superview
// eg make.left.equalTo(@10)
//如果attribute是sizeattribute并且没有第二个attribute
if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
//默认设置item2为superview
secondLayoutItem = self.firstViewAttribute.view.superview;
secondLayoutAttribute = firstLayoutAttribute;
}

//生成约束
MASLayoutConstraint *layoutConstraint
= [MASLayoutConstraint constraintWithItem:firstLayoutItem
attribute:firstLayoutAttribute
relatedBy:self.layoutRelation
toItem:secondLayoutItem
attribute:secondLayoutAttribute
multiplier:self.layoutMultiplier
constant:self.layoutConstant];

//设置priority和mas_key
layoutConstraint.priority = self.layoutPriority;
layoutConstraint.mas_key = self.mas_key;

//如果第二个attribute有view对象
if (self.secondViewAttribute.view) {
//则获取两个view的最小公共view
MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
NSAssert(closestCommonSuperview,
@"couldn't find a common superview for %@ and %@",
self.firstViewAttribute.view, self.secondViewAttribute.view);
//设置约束view为此view
self.installedView = closestCommonSuperview;
} else if (self.firstViewAttribute.isSizeAttribute) {
//如果是size attribute则为本身
self.installedView = self.firstViewAttribute.view;
} else {
//其它则是superview
self.installedView = self.firstViewAttribute.view.superview;
}

//已经存在的约束
MASLayoutConstraint *existingConstraint = nil;
//需要更新
if (self.updateExisting) {
//则获得此生成的约束，返回和installedview的约束是同类的约束
existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
}
//如果存在则替换约束
if (existingConstraint) {
// just update the constant
existingConstraint.constant = layoutConstraint.constant;
//缓存约束类型
self.layoutConstraint = existingConstraint;
} else {
//其它情况则是直接添加约束
[self.installedView addConstraint:layoutConstraint];
//缓存约束类型
self.layoutConstraint = layoutConstraint;
//缓存已经安装约束
[firstLayoutItem.mas_installedConstraints addObject:self];
}
}
```
其中比较有趣的是取得两个view的最小公共view的方法``mas_closestCommonSuperview:``。

至此，简单的Masonry主调用的调用源代码也算全部解析过了。
## 总结
总的来说Masonry的源代码有以下优点:
* 1.大量简洁优美的宏（虽然用宏好不好另说）
* 2.链式调用的实现
* 3.builder建造者模式构成约束类型和设置约束
* 4.NSAssert语句判断类型
* 5.构造抽象类MASConstraint（虽然oc没有抽象类）
* 6.内部接口MASConstraint+Private.h

## 参考资料
1.[Apple-NSLayoutConstraint](https://developer.apple.com/reference/uikit/nslayoutconstraint)
2.[Auto Layout Guide](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/index.html)
3.[Visual Format Language](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html)
3.[深入剖析Auto Layout，分析iOS各版本新增特性](http://www.starming.com/index.php?v=index&view=84&utm_source=tuicool&utm_medium=referral)
4.[iOS开发之Masonry框架源码深度解析](http://www.cnblogs.com/ludashi/p/5591572.html)
5.[Masonry源代码分析](http://blog.csdn.net/colorapp/article/details/45030163)
6.[史上比较用心的纯代码实现 AutoLayout](http://ios.jobbole.com/85829/)
