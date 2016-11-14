####iOS instruments介绍
本文csdn地址：http://blog.csdn.net/game3108/article/details/51147909
写代码的时候，我们时常需要借助一些工具来帮我们分析问题、找到问题，来达到调适和优化代码的目的。在iOS开发方面，XCode提供了一系列工具来帮助我们解决问题，这就是instruments。

**苹果文档这么介绍instruments:**
>Instruments is a powerful and flexible performance-analysis and testing tool that’s part of the Xcode tool set. It’s designed to help you profile your OS X and iOS apps, processes, and devices in order to better understand and optimize their behavior and performance. Incorporating Instruments into your workflow from the beginning of the app development process can save you time later by helping you find issues early in the development cycle.

本文主要介绍一下instruments，和其中几个常用的工具。

######界面介绍

**instruments工作流程图：**
![instruments工作流程图](https://developer.apple.com/library/prerelease/ios/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Art/instruments_workflow_diagram_2x.png)

**打开instruments方法：**

通过Xcode菜单打开instruments:

* Choose Xcode > Open Developer Tool > Instruments

通过Xcode project打开instruments:

* Choose Product > Profile
* Click and hold the Run button in the Xcode toolbar and choose Profile.
* Press Command-I


**instruments主界面图：**
![instruents主界面图](https://developer.apple.com/library/prerelease/ios/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Art/instruments_profilingtemplate_dialog_2x.png)

######Core Animation

>The Core Animation instrument captures information on selected animation statistics. It can record information from a single process or from all processes running on the system.

Core Animation需要注意的一点是，必须是真机调试。
![](http://upload-images.jianshu.io/upload_images/1829891-57b44320c67531fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中调试最主要的以下几个选项：

![](http://upload-images.jianshu.io/upload_images/1829891-08d671c1359b0381.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**以下参考参考链接2:**

**比较重要的：**

* "Color Blended Layers"：图层混合

显示出被混合的图层Blended Layer(用红色标注)，Blended Layer是因为这些Layer是透明的(Transparent)，系统在渲染这些view时需要将该view和下层view混合(Blend)后才能计算出该像素点的实际颜色。所以红色越少越好

* "Color Hits Green and Misses Red":图层缓存

很多视图Layer由于Shadow、Mask和Gradient等原因渲染很高，因此UIKit提供了API用于缓存这些Layer：[layer setShouldRasterize:YES]，系统会将这些Layer缓存成Bitmap位图供渲染使用，如果失效时便丢弃这些Bitmap重新生成。所以绿色越多，红色越少越好

* "Color Offscreen-Rendered Yellow":离屏渲染

Offscreen-Rendering离屏渲染意思是iOS要显示一个视图时，需要先在后台用CPU计算出视图的Bitmap，再交给GPU做Onscreen-Rendering显示在屏幕上，因为显示一个视图需要两次计算，所以这种Offscreen-Rendering会导致app的图形性能下降。所以黄色越少越好。

**次要的：**

* "Color Misaligned Images":图片缩放

Misaligned Image表示要绘制的点无法直接映射到频幕上的像素点，此时系统需要对相邻的像素点做anti-aliasing反锯齿计算，增加了图形负担，通常这种问题出在对某些View的Frame重新计算和设置时产生的。

* "Color Copied images":标注应用绘制时被Core Animation复制的图片
* "Color Immediately":Instruments在做color-flush操作时取消10毫秒的延时
* "Color Compositing Fast-Path Blue":标记由硬件绘制的路径
* "Flash Updated Regions":重绘的区域

对图形性能的分析意义较小，通常仅作为参考。

######Timer Profiler

>The Time Profiler instrument captures stack trace information at prescribed intervals. It can record information from a single process or from all processes running on the system.

![](http://upload-images.jianshu.io/upload_images/1829891-9810dfb0a295de6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于Call Tree的设置参数解释：

**以下参考参考链接4:**

* Separate by Thread: 每个线程应该分开考虑。只有这样你才能揪出那些大量占用CPU的"重"线程  
* Invert Call Tree: 从上倒下跟踪堆栈,这意味着你看到的表中的方法,将已从第0帧开始取样,这通常你是想要的,只有这样你才能看到CPU中话费时间最深的方法.也就是说FuncA{FunB{FunC}} 勾选此项后堆栈以C->B-A 把调用层级最深的C显示在最外面 
* Hide Missing Symbols: 如果dSYM无法找到你的app或者系统框架的话,那么表中看不到方法名只能看到十六进制的数值,如果勾线此项可以隐藏这些符号,便于简化数据
* Hide System Libraries: 勾选此项你会显示你app的代码,这是非常有用的. 因为通常你只关心cpu花在自己代码上的时间不是系统上的
* Show Obj-C Only: 只显示oc代码 ,如果你的程序是像OpenGl这样的程序,不要勾选侧向因为他有可能是C++的  
* Flatten Recursion: 递归函数, 每个堆栈跟踪一个条目


![](http://upload-images.jianshu.io/upload_images/1829891-e41f9599a45e7a30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中可以看到每一个方法的调用时间

双击相应方法，可以看到整个的运行时间：
![](http://upload-images.jianshu.io/upload_images/1829891-dd9d612252125416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


######Leaks
>The Leaks instrument captures information about leaked memory. It can record information from a single process only.

![](http://upload-images.jianshu.io/upload_images/1829891-a6c0996cdd7f4df0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Leaks那行出现的红色标签代表着有内存泄漏。
先在左上角暂停一下程序运行，切换详情的Leaks到Call Tree,
并且在设置界面勾选上"invert Call Tree"和"Hide System  Libraries"，就可以在看到相应的调用函数：

![](http://upload-images.jianshu.io/upload_images/1829891-a9ae1ad89e1b22d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

双击相关函数，就可以跳转到对应出问题的代码，进行修改：

![](http://upload-images.jianshu.io/upload_images/1829891-b37765221e4c73a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####参考链接
1.iOS developer library: [Instruments User Guide](https://developer.apple.com/library/prerelease/ios/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/index.html#//apple_ref/doc/uid/TP40004652-CH3-SW1)
2.iOS App的性能关注点:[iOS App的性能关注点](http://www.hrchen.com/2013/05/performance-with-instruments/)
3.Designing for iOS: Graphics & Performance：[Designing for iOS: Graphics & Performance](https://robots.thoughtbot.com/designing-for-ios-graphics-performance)
4.iOS系类教程之用instruments来检验你的app：[iOS系类教程之用instruments来检验你的app](http://www.cocoachina.com/industry/20140114/7696.html)
5.Instruments Tutorial with Swift: Getting Started：[Instruments Tutorial with Swift: Getting Started](https://www.raywenderlich.com/97886/instruments-tutorial-with-swift-getting-started)
