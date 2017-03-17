## 随手记几个最近碰到的小问题
tips：如果有错误，或者有更好的详细解答，请随时联系我进行修改。

### 1.webview在ATS开启后的问题
虽然苹果推迟了ATS开启的时间，但迟早还是要开启的。
请求都需要HTTPS这个就不多谈了，这边谈一下webview的问题。

有些时候总会用webview去打开一些网站，甚至网站也会跳转一些网站，结果就碰到了ATS被拦截的问题，这个时候怎么解决呢。
1. 关闭ATS，就是在info.plist里``Allow Arbitrary Loads``设置为YES，然后和苹果审核撕逼。
2. 在iOS10上可以使用``NSAllowsArbitraryLoadsInWebContent``和``NSAllowsArbitraryLoadsForMedia``，以让你的 app 中的 UIWebView、WKWebView 或者使用 AVFoundation 播放的在线视频不受 ATS 的限制。
3. 在iOS9以后使用SFSafariViewController去进行加载。问题是接口太少，控制有点难度。
4. 在info.plist里添加``NSExceptionDomains``，一个一个去去除ATS

这里借onevcat的一张图

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1829891-2de12d56b5bc06b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 2.多行输入框问题
UITextField只有一行，没有多行输入。而UITextView可以多行输入，但没有placeholder。
所以要UITextView允许多行输入而且要有placeholder的话，可以在内部添加一个UILabel作为placeholder，然后添加observer观察UITextViewTextDidChangeNotification，在输入变换的时候进行placeholder的UILabel的显示和隐藏。

### 3.WKWebView的弹框
之前把UIWebView升级到WKWebView后发现，比如alert弹框都显示不出来了，查了文档才发现，原来是WKWebView的UIDelegate需要是实现，才能去实现页面内的alert，comfirm，prompt方法，这样也方便app自己内部做自己的相关提示框。如果不实现的话就没有任何弹框。

### 4.关于模块化组件化
以前我在小公司里，就2，3个人开发，所以没有模块化（组件化）这回事，但大公司几百上千个人开发App，这么多的人协作肯定需要各个部门各个模块的化，才能做到互不影响。
可以先看看我写的(iOS架构组件化)[http://www.jianshu.com/p/2d89f55fc2c4]。
常用的就是URL Router的方式。比如蘑菇街的[MGJRouter](https://github.com/mogujie/MGJRouter)就很不错。这里推荐一篇文章，讲得很不错：[组件化架构漫谈](http://www.jianshu.com/p/67a6004f6930)。


## 参考链接：
[CSDN地址](http://blog.csdn.net/game3108/article/details/54838036)
1.[关于 iOS 10 中 ATS 的问题](https://onevcat.com/2016/06/ios-10-ats/)
2.[组件化架构漫谈](http://www.jianshu.com/p/67a6004f6930)
