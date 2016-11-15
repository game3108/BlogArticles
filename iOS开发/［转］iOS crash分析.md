[TOC]
##本文转载于同事的分享文章 siyi.xie，在这里记录一下
 [TOC]

## iOS crash分析 ##
 符号化（symbolicate)
 
 内存地址的解析， 是指从 **`内存地址`** 到 **`符号`**。
 
 ```
 Thread 21 Crashed:
 0   libsystem_kernel.dylib        	0x00000001957b3270 0x195798000 + 111216
 1   libsystem_pthread.dylib       	0x0000000195851224 0x19584c000 + 21028
 2   libsystem_c.dylib             	0x000000019572ab14 0x1956c8000 + 404244
 3   libc++abi.dylib               	0x00000001947fd414 0x1947fc000 + 5140
 4   libc++abi.dylib               	0x000000019481cb88 0x1947fc000 + 134024
 5   libobjc.A.dylib               	0x000000019502c3bc 0x195024000 + 33724
 6   libc++abi.dylib               	0x0000000194819bb0 0x1947fc000 + 121776
 7   libc++abi.dylib               	0x0000000194819474 0x1947fc000 + 119924
 8   libobjc.A.dylib               	0x000000019502c200 0x195024000 + 33280
 9   CoreFoundation                	0x00000001848ce21c 0x1847a8000 + 1204764
 10  TouchPalDialer                	0x000000010052e2c4 0x100028000 + 5268164
 11  TouchPalDialer                	0x000000010052e038 0x100028000 + 5267512
 12  libsystem_platform.dylib      	0x0000000195848948 0x195844000 + 18760
 13  TouchPalDialer                	0x00000001009106b4 0x100028000 + 9340596
 14  TouchPalDialer                	0x00000001009106b4 0x100028000 + 9340596
 15  TouchPalDialer                	0x0000000100918024 0x100028000 + 9371684
 16  TouchPalDialer                	0x00000001009cd354 0x100028000 + 10113876
 17  TouchPalDialer                	0x00000001009ce940 0x100028000 + 10119488
 18  TouchPalDialer                	0x0000000100909a08 0x100028000 + 9312776
 19  TouchPalDialer                	0x00000001009cf698 0x100028000 + 10122904
 20  libsystem_pthread.dylib       	0x000000019584fe7c 0x19584c000 + 15996
 21  libsystem_pthread.dylib       	0x000000019584fdd8 0x19584c000 + 15832
 22  libsystem_pthread.dylib       	0x000000019584cfac 0x19584c000 + 4012
 ```
 
 这里的`内存地址`是指**程序运行时的内存地址**（区别于`程序编译之后的内存地址`）
 
 `符号`是指人类可读的代码信息，一般包括`文件名、类型、函数名、行数`。
 
 比如以下是一行解析后的符号信息：
 
 ```
 1 -[ContactEditNoteView saveNote] (in TouchPalDialer) (ContactEditNoteView.m:98)
 ```
 
 ## 收集crash
 
 ### crash的源头
 **uncaught exception**导致程序终止；程序被终止前，exception能被`uncaught exception hangler`处理。
 
 
 
 > [iOS Developer Library: Uncaught Exceptions][ref-link-ios-dev-uncaught-exception]<br><br>If an exception is not caught, it is intercepted by a function called the uncaught exception handler. **The uncaught exception handler always causes the program to exit but may perform some task before this happens**.
 
 
 **uncaught exception**的来源：
 
 1. 内核
 2. 其他进程
 3. 本身的进程
 
 > [Handling unhandled exceptions and signals][ref-link-coccalove-exception]<br><br>**An unhandled signal can come from three places: the kernel, other processes or the application itself**. The two most common signals that cause crashes are:
 <br>
 > EXC_BAD_ACCESS is a Mach exception sent by the kernel to your application when you try to access memory that is not mapped for your application. If not handled at the Mach level, it will be translated into a SIGBUS or SIGSEGV BSD signal.
 SIGABRT is a BSD signal sent by an application to itself when an NSException or obj_exception_throw is not caught.
 
 
 Mac OS X内核架构如下图所示。mach的exception可能会通过BSD被转换成UNIX类似的signal。
 
 
 ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-8a1968d7498096c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-d99c71676b63ea69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 ### 捕获crash
 
 1. 如果被转成了NSException,可以通过`NSSetUncaughtExecptionHandler()`捕获；
 
 2. 如果以`unix signal`的形式发送给相应的线程，则可以通过C的函数注册对应的信号函数来处理 `signal(int sig, sig_t func)`
 
 
 ### 获取crash的具体信息
 1. `[NSException callStackSymbols];`
 
 2. `int backtrace(void** array, int size);` <br> `char** backtrace_symbols(void* const* array, int size);`
 
 ### 对比
 | 捕获crash的入口 | API  | 获取crash具体信息 | 说明 |
 | --- | --- | --- | --- |
 |  `NSExecption` | `NSSetUncaughtExecptionHandler()`  | `[NSException callStackSymbols]` |    | 
 | `signal`  | `signal(int sig, sig_t func)`  |  int backtrace(void** array, int size); <br> char** backtrace_symbols(void* const* array, int size);   |    |
 
 ### 收集crash的步骤
 1. 注册 `UncaughtExecptionHandler`和`signal(int sig, sig_t func)`
 
 2. 通过对应方法获取 函数调用栈的符号： `[NSException callStackSymbols]` 或者 `backtrace_symbols(void* const* array, int size)`
 
 3. 保存成文件，并上传到服务器
 
 4. 根据调用栈信息和符号表文件（.dSYM文件）进行符号解析。通常使用的命令是：`atos`。比如 `atos -o ./TouchPalDialer.app/TouchPalDialer -arch arm64 -s 0x2760 0x00000001004ed148`
 
 
 ### 收集到的crash实例
 以下是实际代码中收集的crash信息，来自于文件`ios_crash.plist`
 
 ```
 <?xml version="1.0" encoding="UTF-8"?>
 <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
 <plist version="1.0">
 <dict>
	<key>abstract</key>
	<string>UncaughtSignalException: Signal 10 was raised.</string>
	<key>app_name</key>
	<string>com.cootek.Contacts</string>
	<key>app_version</key>
	<string>5384</string>
	<key>detail</key>
	<string>0   TouchPalDialer                      0x00000001005e533c _ZNSbItSt11char_traitsItESaItEE7reserveEm + 378016 |
 1   TouchPalDialer                      0x00000001005e4fd4 _ZNSbItSt11char_traitsItESaItEE7reserveEm + 377144 |
 2   libsystem_platform.dylib            0x0000000197b4094c _sigtramp + 52 |
 3   TouchPalDialer                      0x0000000100a46624 _ZNSt8_Rb_treeISbItSt11char_traitsItESaItEESt4pairIKS3_St6vectorIiSaIiEEESt10_Select1stIS9_ESt4lessIS3_ESaIS9_EE4findERS5_ + 3160 |
 4   TouchPalDialer                      0x0000000100a46624 _ZNSt8_Rb_treeISbItSt11char_traitsItESaItEESt4pairIKS3_St6vectorIiSaIiEEESt10_Select1stIS9_ESt4lessIS3_ESaIS9_EE4findERS5_ + 3160 |
 5   TouchPalDialer                      0x0000000100a462e4 _ZNSt8_Rb_treeISbItSt11char_traitsItESaItEESt4pairIKS3_St6vectorIiSaIiEEESt10_Select1stIS9_ESt4lessIS3_ESaIS9_EE4findERS5_ + 2328 |
 6   TouchPalDialer                      0x0000000100a4bc20 _ZNSt8_Rb_treeIjSt4pairIKjN7orlando7VipInfoEESt10_Select1stIS4_ESt4lessIjESaIS4_EE4findERS1_ + 632 |
 7   TouchPalDialer                      0x00000001005416c0 _ZNSt6vectorISsSaISsEE13_M_insert_auxEN9__gnu_cxx17__normal_iteratorIPSsS1_EERKSs + 666676 |
 8   TouchPalDialer                      0x0000000100540914 _ZNSt6vectorISsSaISsEE13_M_insert_auxEN9__gnu_cxx17__normal_iteratorIPSsS1_EERKSs + 663176 |
 9   libdispatch.dylib                   0x00000001979693ac <redacted> + 24 |
 10  libdispatch.dylib                   0x000000019796936c <redacted> + 16 |
 11  libdispatch.dylib                   0x00000001979734c0 <redacted> + 1216 |
 12  libdispatch.dylib                   0x000000019796c474 <redacted> + 132 |
 13  libdispatch.dylib                   0x0000000197975224 <redacted> + 664 |
 14  libdispatch.dylib                   0x000000019797675c <redacted> + 108 |
 15  libsystem_pthread.dylib             0x0000000197b452e4 _pthread_wqthread + 816 |
 16  libsystem_pthread.dylib             0x0000000197b44fa8 start_wqthread + 4</string>
	<key>device</key>
	<string>iPhone7,1</string>
	<key>manufacturer</key>
	<string>Apple</string>
	<key>os_name</key>
	<string>iOS</string>
	<key>os_version</key>
	<string>iOS 8.1.2</string>
	<key>slide</key>
	<integer>933888</integer>
	<key>timestamp</key>
	<string>1456103577</string>
 </dict>
 </plist>
 
 ```
 
 ## 工具: `atos`
 ### 关键：`load address`, `slide`
 
 ```
 atos -- convert numeric addresses to symbols of binary images or processes
 
 atos [-o <binary-image-file>] [-p <pid> | <partial-executable-name>] [-arch architecture]
 [-l <load-address>] [-s <slide>] [-printHeader] [-f <address-input-file>]
 [<address> ...]
 
 ```
 
 使用`atos`使用举例：
 
 ```
 atos -o ./TouchPalDialer.app/TouchPalDialer -arch arm64 -s 0x2760 0x00000001004ed148
 ```
 
 **注意：**
 
 命令`atos`中输入的解析地址要求是`构建的内存地址（built address）`（因为这个地址是要对应于dSYM文件的，而dSYM文件是在构建产生的），而获取并上传的函数调用栈的地址是程序在运行时的内存地址，称之为`实际加载地址`。
 
 
 ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-f8b12bd380e9aed4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 [苹果开发者文档(Technical Note TN2123)][ref-technote-crash-reporter]指出，需要区分两个加载地址（load address）：`实际加载地址` `期望加载地址`
 
 1. **`实际加载地址 (actual load address)`** 
 是指程序运行时的地址。每次程序装载(load)都可能不同。
 
 2. **`期望加载地址 (intented load address)`** 
 是指编译之后指定的加载地址。
 
 ` 偏移量 = 实际加载地址 -  期望加载地址 `
 
 ` slide = acutal load address - intended load addrss `
 
 
 
 ### 获取`load address`,`slide`
 + **加载地址(load address)**<br>如果能够获取有系统的CrashReporter产生的完成的崩溃日志文件(.crash文件)，则从其中的`Binary Images`部分获取当前程序的`**实际加载地址(acutal load address)**`
 
 举例如下：
 
 ```
 Binary Images:
 0x100028000 - 0x100c53fff TouchPalDialer arm64  <a197140a05563a0c983ef71bb8a83e1b> /var/mobile/Containers/Bundle/Application/C5B024A9-A8CC-4F1D-8613-99376B830C3E/TouchPalDialer.app/TouchPalDialer
 0x1200c0000 - 0x1200e7fff dyld arm64  <36eff49275c23d2d815e48af33eea471> /usr/lib/dyld
 0x182e60000 - 0x182e7dfff libJapaneseConverter.dylib arm64  <39642fdaade73029adb01e10922d2ba3> /System/Library/CoreServices/Encodings/libJapaneseConverter.dylib
 ```
 
 + **偏移量(slide)** <br>调用底层C的API获取，`_dyld_get_image_vmaddr_slide(uint32_t image_index)`
 
 举例如下：
 
 ```
 + (long) getImageSlide {
 long slide = -1;
 for (uint32_t i = 0; i < _dyld_image_count(); i++) {
 if (_dyld_get_image_header(i)->filetype == MH_EXECUTE) {
 slide = _dyld_get_image_vmaddr_slide(i);
 break;
 }
 }
 return slide;
 }
 ```
 
 ### 参考资料
 
 + How to read objective-c stack traces
 http://stackoverflow.com/questions/6462214/how-to-read-objective-c-stack-traces
 
 + Technical Note TN2151
 Understanding and Analyzing iOS Application Crash Reports
 https://developer.apple.com/library/ios/technotes/tn2151/_index.html
 
 + Technical Note TN2123
 CrashReporter
 https://developer.apple.com/library/mac/technotes/tn2004/tn2123.html
 
 + Demystifying iOS Application Crash Logs
 http://www.raywenderlich.com/23704/demystifying-ios-application-crash-logs
 
 + Symbolicating Your iOS Crash Reports
 https://possiblemobile.com/2015/03/symbolicating-your-ios-crash-reports/
 
 + Exploring iOS Crash Reports
 <br>https://www.plausible.coop/blog/?p=176
 
 + `intended load address` vs `actual load address`
 http://www.cocoabuilder.com/archive/xcode/312725-can-symbolicate-crash-reports.html
 
 + tools `man atos`
 https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/atos.1.html
 
 + `man otool`
 <br>http://www.unix.com/man-page/OSX/1/otool/
 
 + Tales From The Crash Mines: Issue #1, by Landon Fuller
 https://www.mikeash.com/pyblog/tales-from-the-crash-mines-issue-1.html
 
 + 你的iOS安装包真的“脱光”了么
 <br>http://zhuanlan.zhihu.com/geek-makes-life-better/19925959
 
 + 使用Symbolicatecrash和xcrun atos分析crash log
 http://blog.csdn.net/mkhgg/article/details/17247673
 
 + stackoverflow
 How to read objective-c stack traces
 http://stackoverflow.com/questions/6462214/how-to-read-objective-c-stack-traces
 
 + iOS crash reports: atos not working as expected
 http://stackoverflow.com/questions/13574933/ios-crash-reports-atos-not-working-as-expected
 
 
 + `dwarfdump`: 关于XCode编译完App之后生成的dSYM文件
 http://blog.csdn.net/yan8024/article/details/8186760
 
 + 使用dwarfdump检查dSYM和app是否匹配
 http://blog.csdn.net/yan8024/article/details/8186774
 
 + load address 
 Can't symbolicate crash reports
 http://www.cocoabuilder.com/archive/xcode/312725-can-symbolicate-crash-reports.html
 
 + Debugging stack traces
 <br>http://www.cocoabuilder.com/archive/cocoa/276690-debugging-stack-traces.html
 
 + static or dynamic libraries Dynamic linking on iOS
 http://ddeville.me/2014/04/dynamic-linking/
 
 + The LLDB Debugger
 <br>http://lldb.llvm.org/symbolication.html
 
 
 
 ## 补充资料
 
 ### memory layout 内存布局
 虚拟内存地址，从`低地址`到`高地址`
 
 1. Text segment
 2. Initialized data segment
 3. Uninitialized data segment
 4. Stack
 5. Heap
 
 
 ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-34dc0efda79e9267.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 
 ### 函数调用栈 callstack
 
 函数`压栈 push`, 函数`退栈 pop`
 
 
 ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-ef786434f8bbc66a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 
 ### 实际代码：`TPUncaughtExceptionHandler.m`
 
 ``` 
 + (void) attachHandler {
 NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler); 
 for(int i = 0; i< signalCounts; i++) {
 signal(monitorSignals[i], SignalHandler__);
 }
 }
 
 // The signal handler for bad signls defined in monitorSignals array
 // Gather callstack, convert to exception and then use the same exception handling approach.
 void SignalHandler__(int signal) {
 NSMutableDictionary *info =
 [NSMutableDictionary
 dictionaryWithObject:[NSNumber numberWithInt:signal]
 forKey:SignalKey];
 
	NSArray *callStack = [TPUncaughtExceptionHandler backtrace];
 
	[info setObject:callStack forKey:CallStackKey];
 [info setObject:@([TPUncaughtExceptionHandler getImageSlide]) forKey:SlideKey];
	
 [TPUncaughtExceptionHandler handleExceptionAndExit:
 [NSException
 exceptionWithName:UncaughtSignalExceptionName
 reason:[NSString stringWithFormat:@"Signal %d was raised.", signal]
 userInfo:info]];
 }
 
 + (NSArray *)backtrace
 {
 const int maxCount = 128;
 void* callstack[maxCount];
 int count = backtrace(callstack, maxCount);
 char **strs = backtrace_symbols(callstack, count);
 
 NSMutableArray *backtrace = [NSMutableArray arrayWithCapacity:count];
 for (int i = 0; i < count; i++)
 {
 [backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
 }
 free(strs);
 
 return backtrace;
 }
 
 /**
 *
 *  @return the slide of this binary image
 */
 + (long) getImageSlide {
 long slide = -1;
 for (uint32_t i = 0; i < _dyld_image_count(); i++) {
 if (_dyld_get_image_header(i)->filetype == MH_EXECUTE) {
 slide = _dyld_get_image_vmaddr_slide(i);
 break;
 }
 }
 cootek_log(@"TPUncaughtExceptionHandler, slide: %s", @(slide).description);
 return slide;
 }
 
 ```
 
 ### 通过backtrace获取的crash信息
 ```
 0 TouchPalDialer 0x00000001004ff418 _ZNSbItSt11char_traitsItESaItEE7reserveEm + 370100 |
 1 TouchPalDialer 0x00000001004ff154 _ZNSbItSt11char_traitsItESaItEE7reserveEm + 369392 |
 2 libsystem_platform.dylib 0x000000019506094c _sigtramp + 52 |
 3 TouchPalDialer 0x00000001008e5ed4 _ZN15CTXAppidConvert17IsConnectionAppIdEPKc + 720196 |
 4 TouchPalDialer 0x00000001008e5ed4 _ZN15CTXAppidConvert17IsConnectionAppIdEPKc + 720196 |
 5 TouchPalDialer 0x00000001008dacb4 _ZN15CTXAppidConvert17IsConnectionAppIdEPKc + 674596 |
 6 TouchPalDialer 0x00000001008e2098 _ZN15CTXAppidConvert17IsConnectionAppIdEPKc + 704264 |
 7 TouchPalDialer 0x00000001009973c8 _ZNSt8_Rb_treeIiSt4pairIKiPN7orlando12YellowSearchEESt10_Select1stIS5_ESt4lessIiESaIS5_EE4findERS1_ + 223600 |
 8 TouchPalDialer 0x00000001009989b4 _ZNSt8_Rb_treeIiSt4pairIKiPN7orlando12YellowSearchEESt10_Select1stIS5_ESt4lessIiESaIS5_EE4findERS1_ + 229212 |
 9 TouchPalDialer 0x00000001008d3a7c _ZN15CTXAppidConvert17IsConnectionAppIdEPKc + 645356 |
 10 TouchPalDialer 0x0000000100999730 _ZNSt8_Rb_treeIiSt4pairIKiPN7orlando12YellowSearchEESt10_Select1stIS5_ESt4lessIiESaIS5_EE4findERS1_ + 232664 |
 11 libsystem_pthread.dylib 0x0000000195067dc8 <redacted> + 164 |
 12 libsystem_pthread.dylib 0x0000000195067d24 <redacted> + 0 |
 13 libsystem_pthread.dylib 0x0000000195064ef8 thread_start + 4
 ```
 
 
 
 [ref-link-crash-collector-framework]: http://www.cocoachina.com/ios/20150701/12301.html "漫谈iOS Crash收集框架"
 
 [ref-link-ios-dev-uncaught-exception]: https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Exceptions/Concepts/UncaughtExceptions.html "Uncaught Exceptions"
 
 [ref-technote-crash-reporter]: https://developer.apple.com/library/mac/technotes/tn2004/tn2123.html "Technical Note TN2123 CrashReporter" 
 
 [ref-link-coccalove-exception]: http://www.cocoawithlove.com/2010/05/handling-unhandled-exceptions-and.html "Handling unhandled exceptions and signals"
 
 [ref-img-c-memory-layout]: /content/images/2016/02/memory-layout-c.gif "Memory Layout of C Programs, http://www.geeksforgeeks.org/memory-layout-of-c-program/"
 
 [ref-img-mac-kernal]: /content/images/2016/02/osxarchitecture-1.gif "Mac OS X kernel architecture, http://docs.huihoo.com/darwin/kernel-programming-guide/Architecture/chapter_3_section_3.html"
 
 [ref-img-mac-arch-illustration]: /content/images/2016/02/mac-arch-2-1.png "Mac OS X Tiger, http://informaticaitaliana.blogspot.com/2014/06/mac-os-x-tiger.html"
 
 
 [ref-img-binary-slide]: /content/images/2016/02/slide-load-address-1.png "slide, load address"
