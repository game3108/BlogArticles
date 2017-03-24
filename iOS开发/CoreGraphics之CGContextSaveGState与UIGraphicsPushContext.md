## 前言
写这文章的原因是最近在写CG的时候，对于CGContextSaveGState与UIGraphicsPushContext的区别感到有一些困惑，就做了一些试验在这里列出来。

## CoreGraphics与UIKit
这边从[iOS绘图教程](http://www.cnblogs.com/xdream86/archive/2012/12/12/2814552.html) 提取一些重要的内容。

Core Graphics Framework是一套基于C的API框架，使用了Quartz作为绘图引擎。iOS支持两套图形API族：Core Graphics/QuartZ 2D 和OpenGL ES。

Core Graphics API所有的操作都在一个上下文中进行。所以在绘图之前需要获取该上下文并传入执行渲染的函数中。如果你正在渲染一副在内存中的图片，此时就需要传入图片所属的上下文。获得一个图形上下文是我们完成绘图任务的第一步，你可以将图形上下文理解为一块画布。如果你没有得到这块画布，那么你就无法完成任何绘图操作。当然，有许多方式获得一个图形上下文，这里我介绍两种最为常用的获取方法。
 
1. 创建一个图片类型的上下文。调用``UIGraphicsBeginImageContextWithOptions``函数就可获得用来处理图片的图形上下文。利用该上下文，你就可以在其上进行绘图，并生成图片。调用``UIGraphicsGetImageFromCurrentImageContext``函数可从当前上下文中获取一个UIImage对象。记住在你所有的绘图操作后别忘了调用``UIGraphicsEndImageContext``函数关闭图形上下文。

2. 利用cocoa为你生成的图形上下文。当你子类化了一个UIView并实现了自己的``drawRect：``方法后，一旦``drawRect：``方法被调用，Cocoa就会为你创建一个图形上下文，此时你对图形上下文的所有绘图操作都会显示在UIView上。
 
判断一个上下文是否为当前图形上下文需要注意的几点：
1. ``UIGraphicsBeginImageContextWithOptions``函数不仅仅是创建了一个适用于图形操作的上下文，并且该上下文也属于当前上下文。
2. 当``drawRect``方法被调用时，UIView的绘图上下文属于当前图形上下文。
3. 回调方法所持有的context：参数并不会让任何上下文成为当前图形上下文。此参数仅仅是对一个图形上下文的引用罢了。
 
作为初学者，很容易被UIKit和Core Graphics两个支持绘图的框架迷惑。
 
#### UIKit
像UIImage、NSString（绘制文本）、UIBezierPath（绘制形状）、UIColor都知道如何绘制自己。这些类提供了功能有限但使用方便的方法来让我们完成绘图任务。一般情况下，UIKit就是我们所需要的。
 
使用UiKit，你只能在当前上下文中绘图，所以如果你当前处于``UIGraphicsBeginImageContextWithOptions``函数或``drawRect：``方法中，你就可以直接使用UIKit提供的方法进行绘图。如果你持有一个context：参数，那么使用UIKit提供的方法之前，必须将该上下文参数转化为当前上下文。幸运的是，调用``UIGraphicsPushContext`` 函数可以方便的将context：参数转化为当前上下文，记住最后别忘了调用UIGraphicsPopContext函数恢复上下文环境。
 
#### Core Graphics
这是一个绘图专用的API族，它经常被称为QuartZ或QuartZ 2D。Core Graphics是iOS上所有绘图功能的基石，包括UIKit。
 
使用Core Graphics之前需要指定一个用于绘图的图形上下文（CGContextRef），这个图形上下文会在每个绘图函数中都会被用到。如果你持有一个图形上下文context：参数，那么你等同于有了一个图形上下文，这个上下文也许就是你需要用来绘图的那个。如果你当前处于``UIGraphicsBeginImageContextWithOptions``函数或``drawRect：``方法中，并没有引用一个上下文。为了使用Core Graphics，你可以调用``UIGraphicsGetCurrentContext``函数获得当前的图形上下文。

##stackoverflow的问题
在stackoverflow上，有这样一个问题[CGContextSaveGState vs UIGraphicsPushContext](http://stackoverflow.com/questions/15505871/cgcontextsavegstate-vs-uigraphicspushcontext)问了两者区别，这里列一下高票答案：
>UIGraphicsPushContext(context) pushes context onto a stack of CGContextRefs (making context the current drawing context), whereas CGContextSaveGState(context) pushes the current graphics state onto the stack of graphics states maintained by context. You should use UIGraphicsPushContext if you need to make a new CGContextRef the current drawing context, and you should use CGContextSaveGState when you're working with one graphics context and just want to save, for example: the current transform state, fill or stroke colors, etc.

翻译一下就是：
``UIGraphicsPushContext(context)``将context压到一个CGContextRefs(使得context成为current context)的栈中。而``CGContextSaveGState(context)``将当前绘制状态压到一个context维护的绘制状态的栈中。你可以使用``UIGraphicsPushContext``当你需要在当前的context去创建一个新的CGContextRef，同时你可以使用``CGContextSaveGState``当你在处理一个绘制context并且只是想保存的它的时候。比如：当前的变换状态，填充或者线条颜色等。

以上答案其实就是在说：
1. ``UIGraphicsPushContext``:压栈当前的绘制对象，生成新的绘制图层
2. ``CGContextSaveGState``:压栈当前的绘制状态

## 实例

#### CGContextSaveGState
我们这里用一段实际代码：
```
-(void)drawRect:(CGRect)rect{

    CGContextRef ctx=UIGraphicsGetCurrentContext();
    [[UIColor redColor] setStroke];                                 //红色
    
    CGContextSaveGState(UIGraphicsGetCurrentContext());
    
    CGContextAddEllipseInRect(ctx, CGRectMake(100, 100, 50, 50));
    CGContextSetLineWidth(ctx, 10);
    [[UIColor yellowColor] setStroke];                               //黄色
    CGContextStrokePath(ctx);
    
    CGContextRestoreGState(UIGraphicsGetCurrentContext());
    
    CGContextAddEllipseInRect(ctx, CGRectMake(200, 100, 50, 50));    //红色
    CGContextStrokePath(ctx);
}
```
运行一下看结果：

![结果图](http://upload-images.jianshu.io/upload_images/1829891-6cd813ff53c2cbad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**可以看到，``CGContextSaveGState``存储下来了当前红色和默认的线条状态，然后切换颜色到黄色和10粗度的线条画圈，然后在``CGContextRestoreGState``恢复到了红色和默认的线条状态进行画圈，这个就是存储当前绘制状态的意思。**

####UIGraphicsPushContext
同样用一段实际代码：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    CALayer *layer=[CALayer layer];
    layer.bounds=CGRectMake(0, 0, 300, 300);
    layer.position=CGPointMake(100, 100);
    layer.delegate=self;
    [layer setNeedsDisplay];
    [self.view.layer addSublayer:layer];
}

-(void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx{
    UIImage *image = [UIImage imageNamed:@"test.jpg"];
    
    UIGraphicsPushContext(ctx);
    [image drawInRect:CGRectMake(0, 0, 300, 300)];
    UIGraphicsPopContext();
}
```
运行看一下结果：

![结果图2](http://upload-images.jianshu.io/upload_images/1829891-22ce998fa3863c44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你将``UIGraphicsPushContext(ctx);``与``UIGraphicsPopContext();``删去的话，是无法进行绘制的。

** 原因是，UIKit的绘制必须在当前的上下文中绘制，而UIGraphicsPushContext可以将当前的参数context转化为可以UIKit绘制的上下文，进行绘制图片。**

## 总结
``CGContextSaveGState``是压栈当前的绘制状态，而``UIGraphicsPushContext``:压栈当前的绘制对象，生成新的绘制图层。对于``UIGraphicsPushContext``的使用，很多都是与UIKit配合使用，更详细的对于CoreGraphics的介绍，可以参考[iOS绘图教程](http://www.cnblogs.com/xdream86/archive/2012/12/12/2814552.html) 。

## 参考资料
[本文CSDN地址](http://blog.csdn.net/game3108/article/details/54576633)
1.[CGContextSaveGState vs UIGraphicsPushContext](http://stackoverflow.com/questions/15505871/cgcontextsavegstate-vs-uigraphicspushcontext)
2.[iOS --- CoreGraphics中三种绘图context切换方式的区别](http://icetime17.github.io/2015/12/29/2015-12/iOS-CoreGraphics%E4%B8%AD%E4%B8%89%E7%A7%8D%E7%BB%98%E5%9B%BEcontext%E5%88%87%E6%8D%A2%E6%96%B9%E5%BC%8F%E7%9A%84%E5%8C%BA%E5%88%AB/?utm_source=tuicool&utm_medium=referral)
3.[iOS core graphic使用分析](http://blog.csdn.net/zhengyueyang71104233/article/details/15335683)
4.[iOS绘图教程](http://www.cnblogs.com/xdream86/archive/2012/12/12/2814552.html) | [iOS绘图教程](http://www.cocoachina.com/industry/20140115/7703.html)
