## 前言
这个系列就是随手记录一下平时捧到的一些问题和一些尝试。

本文csdn地址：http://blog.csdn.net/game3108/article/details/52302610

tips：如果有错误，或者有更好的详细解答，请随时联系我进行修改。

## 解决的问题
### 1.statusbar颜色问题

>info.plist文件中，“View controller-based status bar appearance”项设为YES，则View controller对status bar的设置优先级高于application的设置。为NO则以application的设置为准，view controller的prefersStatusBarHidden方法无效，是根本不会被调用的。

意思就是``View controller-based status bar appearance``为YES：
```
[[UIApplication sharedApplication]*setStatusBarStyle*:UIStatusBarStyleLightContent];
```
无效，只能使用UIViewController的方法
```
- (UIStatusBarStyle)preferredStatusBarStyle
{
    return UIStatusBarStyleLightContent;
}
```
设为no则反过来。

另外，当UIViewController自定义navigationController，并且navigationController同样继承该方法，UIViewController该方法无效。在navigationController中定义下面方法，则uiviewcontroller有效
```
- (UIViewController *)childViewControllerForStatusBarStyle {
    return [self.childViewControllers lastObject];
}
```

### 2.webview NSURLErrorServerCertificateUntrusted 问题
webview加载某些页面，证书不可信，报1202error code，可用如下方法:
```
#pragma mark UIWebViewDelegate
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
{
    NSString *url = request.URL.absoluteString;
    if (url.length == 0 || ![url hasPrefix:@"http"])
        return NO;
    _request = request;
    NSLog(@"paypal web url : %@",url);
    return YES;
}

- (void)webView:(UIWebView *)webView didFailLoadWithError:(nullable NSError *)error{
    if ( error.code == NSURLErrorServerCertificateUntrusted ){
        _urlConnection = [[NSURLConnection alloc] initWithRequest:_request delegate:self];
        [_urlConnection start];
    }
}

#pragma mark - NURLConnection delegate

- (void)connection:(NSURLConnection *)connection didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
{
    if ([challenge previousFailureCount] == 0)
    {
        NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        [challenge.sender useCredential:credential forAuthenticationChallenge:challenge];
    } else
    {
        [[challenge sender] cancelAuthenticationChallenge:challenge];
    }
}

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response
{
    [_urlConnection cancel];
    _urlConnection = nil;
    [_webView loadRequest:_request];
}

// We use this method is to accept an untrusted site which unfortunately we need to do, as our PVM servers are self signed.
- (BOOL)connection:(NSURLConnection *)connection canAuthenticateAgainstProtectionSpace:(NSURLProtectionSpace *)protectionSpace
{
    return [protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust];
}
```

### 3.ui小细节
* 1.对于uiviewcontroller，ui初始化位置，``viewdidload``好过``init``，因为会有self.view的大小问题。
* 2.``autoresizingMask``适合给cell使用，或者是类似初始化读self.view之类的宽度读出来是默认320的地方。
* 3.parentview有``autoresizingMask``变化，但子view不会跟着变化。所以很多系统ui包含子view的，需要注意一下（比如uiwebview）。
* 4.在iOS 7中，苹果引入了一个新的属性，叫做[UIViewController setEdgesForExtendedLayout:]，它的默认值为UIRectEdgeAll。当你的容器是navigation controller时，默认的布局将从navigation bar的顶部开始。这就是为什么所有的UI元素都往上漂移了44pt。
修复这个问题的快速方法就是在方法- (void)viewDidLoad中添加如下一行代码：
self.edgesForExtendedLayout = UIRectEdgeNone;
* 5.UILabel的``sizeToFit``和``autoresizingMask = UIViewAutoresizingFlexibleWidth``可能存在冲突。

### 4.iOS10的无限layoutsubviews问题
iOS10下，在cell中使用``layoutsubviews``，中如果2次设置cell的frame，并且2次frame有不同，就会无限调用``layoutsubviews``，从而卡死。
其他iOS系统没这问题，目测是iOS10测试版本的bug。

### 5.autoresizingMask与layoutsubviews生效时机问题
autoresizingMask先生效，然后生效layoutsubviews。

### 6.VIEW DEBUG时候，碰到的_UIView系列都是系统的UI
比如tableview出现的_UITableViewSeparatorView

### 7.UILabel sizeToFit会根据当前的view width进行fit，所以第一次很短的text会导致uilabel的view width变小，这个时候如果有一个长text，要重新设置view width再sizeToFit

### 11.8更新
8.系统默认uitableview如果不去掉分割线，separateView会有0.5dp(1px)的高度。所以cell内的contentview的高度会差cell的高度0.5dp。

## 未解决的问题
未解决的问题，如果有人知晓答案的话，麻烦评论指点一下。
1.key不存在在strings文件中，NSLocalizedStringFromTable未找到key的时候，返回的key会从小写变成大写
key不存在在strings文件中，用NSLocalizedStringFromTable去寻找某个key。
strings文件中所有key为小写，但在某些分支上，返回值会变成key的大写。但在其他分支上不会。

## 参考链接：
1.[[iOS]关于状态栏(UIStatusBar)的若干问题](http://www.cnblogs.com/alby/p/4859537.html)
2.[stackoverflow](http://stackoverflow.com/questions/11573164/uiwebview-to-view-self-signed-websites-no-private-api-not-nsurlconnection-i)
