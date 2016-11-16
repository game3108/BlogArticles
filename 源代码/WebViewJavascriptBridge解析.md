#### 前序
本文csdn地址：http://blog.csdn.net/game3108/article/details/51147923
iOS原生应用和web页面的交互主要有：JavaScriptCore（iOS7以后）与拦截协议两个方法。

因为我们的app要兼容iOS6,所以我们在web js和native交互使用的是拦截协议的一个很有名的第三方框架：**WebViewJavascriptBridge**，本文从源代码来解析一下WebViewJavascriptBridge的工作方式。
（对于iOS8新出的**WKWebView**，原理方式相同，会稍作提及，但不会详细展开。）

#### iOS web和native交互的方式
首先抛开WebViewJavascriptBridge,思考一个问题，如果我们自己去做一套js与native交互的轮子，应该如何去做？

* 寻找js与native可能交互的接口
* 设计一套融合交互接口的数据模式
* 完善整体轮子

我们来依此分析每一项

###### 寻找js与native可能交互的接口

查询相应的文档可以得知，在UIWebView中，native有直接调用js的方法,js没有直接调用native的方法

native直接调用js的方法：
```objectivec
- (nullable NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;

- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ __nullable)(__nullable id, NSError * __nullable error))completionHandler;//WKWebView使用，以下类推)
```
那对于js来说，无法直接调用native代码是否表示无法进行交互？

**答案是否定的。虽然无法直接调用native代码，但是iOS的接口中还是设计了可以通过间接的方式传递js调用的消息**

js间接调用native的方式：
对于iOS UIWebView熟悉的人来说，

```objectivec
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
```

这个方法，可以每次在UIWebView进行重定向URL的时候，进行触发，只要把一个js调用native的方法包装成一个重定向URL,就可以在本地接收到相应的方法。
```objectivec
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler;
```

**小结：
native可以直接调用js,js将方法包装成重定向请求，使得native截取并分析，执行相应代码。**
问题：那就是说，js代码每次把一次调用里面所有参数都集合进一个重定向请求里直接让本地截取和执行？

###### 设计一套融合交互接口的数据模式

在js与native交互的时候，设计一套两边都可以使用的模式十分关键。

native代码与交互js代码明显是异步的一个操作，尤其是在双方初始化的时候，native代码无法确定js代码是否加载完成，如果在js未加载完成的时候进行相应方法调用，是没有效果的。
所以在双方交互的时候，设计一个list数组，去存储两遍在未初始化完成前需要执行的方法，十分重要。

而对于每一个方法请求，我们至少需要以下参数：

* 方法名
* 方法参数
* 回调

在此基础上，key-value对（Dictionary）会是一个很好的选择。

**小结：交互设计上，两遍都会初始化一个list去存储未初始化时候对方的方法请求。
而对于每一个方法请求，通过key-value对，去提供双方解析。**

###### 完善整体轮子

有了交互的接口，还有数据格式，双方获取到对方的方法调用进行处理就不再困难。
在其中，唯一还有些绕的是双方的回调。
js调用native的回调，可以在native中直接通过调用js的方式，进行回调函数的调用。
而native调用js的回调，还是要通过native截取js的方式，进行回调函数的调用。
所以双方都需要把本地的回调函数通过key-value对(Dictionary)存储下来。

那在整体设计上，双方的方法解析与互相调用也应该分离开来，由此可以达到代码模块化的目的。

**小结：在完善轮子的时候，根据设计模式的原则，进行相应的模块化，方便代码复用。**

#### 解析WebViewJavascriptBridge的源代码

上一章我们整体设计了我们自己的一个native与js交互的轮子，WebViewJavascriptBridge本身的做法也是类似，现在我们解析WebViewJavascriptBridge的源代码来了解它是如何做到这每一步。
我们将js调用native的整个流程走一遍，就可以完全清楚WebViewJavascriptBridge的逻辑。

首先是在页面加载完成的时候，native会注入一段js代码：
```objectivec
- (void)webViewDidFinishLoad:(UIView<FLWebViewProvider> *)webView {
    if (webView != _webView) { return; }
    _numRequestsLoading--;
    if (_numRequestsLoading == 0 && ![[(UIWebView *)webView stringByEvaluatingJavaScriptFromString:[_base webViewJavascriptCheckCommand]] isEqualToString:@"true"]) {
        [_base injectJavascriptFile:YES webView:webView];    }
    [_base dispatchStartUpMessageQueue];
    __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
    if (strongDelegate && [strongDelegate respondsToSelector:@selector(webViewDidFinishLoad:)]) {
        [strongDelegate webViewDidFinishLoad:(UIWebView *)webView];
    }
}
```

其中
```objectivec
[_base injectJavascriptFile:YES webView:webView]

- (void)injectJavascriptFile:(BOOL)shouldInject webView:(UIView<FLWebViewProvider> *)webview {
    if(shouldInject){
        NSBundle *bundle = _resourceBundle ? _resourceBundle : [NSBundle mainBundle];
        NSString *filePath = [bundle pathForResource:@"WebViewJavascriptBridge.js" ofType:@"txt"];
        NSString *jsBridge = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
        [self webview:webview evaluateJavaScript:jsBridge completionHandler:^(id callback, NSError *error) {
            [self injectForDispatch:webview];
        }];
    }
}
```
将本地端的WebViewJavascriptBridge.js.txt注入到了web页面的js代码中


WebViewJavascriptBridge初始化时候，本身提供了ExampleApp.html页面
我们从项目中找处一句js调用native的语句：

```objectivec
bridge.callHandler('testObjcCallback', {'foo': 'cccccccccccc'}, function(response) {  
	log('JS got response', response)  
})
```

我们找到WebViewJavascriptBridge.js.txt

```javascript
function callHandler(handlerName, data, responseCallback) {
		_doSend({ handlerName:handlerName, data:data }, responseCallback)
	}
function _doSend(message, responseCallback) {
		if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime()
			responseCallbacks[callbackId] = responseCallback
			message['callbackId'] = callbackId
		}
		sendMessageQueue.push(message)
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE
	}
```

其中，正如我们自己设计的，
首先，我们会将请求分为3个部分

* handlerName	方法名
* data		方法参数
* callback	回调

用dictionary的方式存储他们，组成了message，message本身就是我们发起的请求
而responseCallbacks通过特殊的callbackid，存储下了这一次的函数回调
message增加了一个’callbackid’的参数，去存储这个callbackid

上面都是我们设计的方式，就是比较特殊的地方是，
WebViewJavascriptBridge将整个请求message，塞入了sendMessageQueue中，而并非我们想当然的塞入url重定向中。
然后使用messagingIframe发起重定向。

```javascript
function _createQueueReadyIframe(doc) {
		messagingIframe = doc.createElement('iframe')
		messagingIframe.style.display = 'none'
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE
		doc.documentElement.appendChild(messagingIframe)
	}
var CUSTOM_PROTOCOL_SCHEME = 'wvjbscheme'
var QUEUE_HAS_MESSAGE = '__WVJB_QUEUE_MESSAGE__'
```

而这个重定向url由CUSTOM_PROTOCOL_SCHEME和QUEUE_HAS_MESSAGE组成。

从这里可以看出，js调用本地代码，并没有直接将参数请求塞入重定向url，而是塞入了一个list中，而所有的参数请求都是发了同一个。

再看本地代码，之前所说的解析代码：

本地 WebViewJavascriptBridge.m

```objectivec
- (BOOL)webView:(UIView<FLWebViewProvider> *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    if (webView != _webView) { return YES; }
    NSURL *url = [request URL];
    __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
    if ([_base isCorrectProcotocolScheme:url]) {
        if ([_base isCorrectHost:url]) {
            NSString *messageQueueString = [(UIWebView*)webView stringByEvaluatingJavaScriptFromString:[_base webViewJavascriptFetchQueyCommand]];
            [_base flushMessageQueue:messageQueueString];
        } else {
            [_base logUnkownMessage:url];
        }
        return NO;
    } else if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
        return [strongDelegate webView:(UIWebView *)webView shouldStartLoadWithRequest:request navigationType:navigationType];
    } else {
        return YES;
    }
}
```

其中_base是一个WebViewJavascriptBridgeBase对象,它包含了：

```objectivec
-(BOOL)isCorrectProcotocolScheme:(NSURL*)url {
    if([[url scheme] isEqualToString:kCustomProtocolScheme]){
        return YES;
    } else {
        return NO;
    }
}
-(BOOL)isCorrectHost:(NSURL*)url {
    if([[url host] isEqualToString:kQueueHasMessage]){
        return YES;
    } else {
        return NO;
    }
}
```
```objectivec
#define kCustomProtocolScheme @"wvjbscheme"

#define kQueueHasMessage @"__WVJB_QUEUE_MESSAGE__"
```

其中，很显然，通过检查scheme和host，就可以清楚的知道，这个请求是不是WebViewJavascriptBridge的重定向请求。
而

```objectivec
NSString *messageQueueString = [(UIWebView*)webView stringByEvaluatingJavaScriptFromString:[_base webViewJavascriptFetchQueyCommand]];
```

很明显是在获得之前sendMessageQueue

查找WebViewJavascriptBridgeBase.m和WebViewJavascriptBridge.js.txt文件

```objectivec
-(NSString *)webViewJavascriptFetchQueyCommand {
    return @"WebViewJavascriptBridge._fetchQueue();";
}
function _fetchQueue() {
    var messageQueueString = JSON.stringify(sendMessageQueue)
    sendMessageQueue = []
    return messageQueueString
}
```

那这边我们就了解了js调用本地端的方法：
js发起一个特殊的url请求，告诉本地端，我发起请求了。并且存储相应的回调函数和消息队列。
而本地端接收到消息，会去主动拉取js中存储消息的队列。

之后就是处理消息的方式：

```objectivec
[_base flushMessageQueue:messageQueueString];
```

找到WebViewJavascriptBridgeBase.m

```objectivec
- (void)flushMessageQueue:(NSString *)messageQueueString{
    id messages;
    if (messageQueueString) {
        messages = [self _deserializeMessageJSON:messageQueueString];
    }
    if (![messages isKindOfClass:[NSArray class]]) {
        NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [messages class], messages);
        return;
    }
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];
        NSString* responseId = message[@"responseId"];
        if (responseId) {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else {
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            WVJBHandler handler;
            if (message[@"handlerName"]) {
                handler = self.messageHandlers[message[@"handlerName"]];
            } else {
                handler = self.messageHandler;
            }
            if (!handler) {
//                [NSException raise:@"WVJBNoHandlerException" format:@"No handler for message from JS: %@", message];
                cootek_log(@"No handler for message from JS: %@",message);
            } else {
                handler(message[@"data"], responseCallback);
            }
        }
    }
}
```

首先，解析messages

```objectivec
messages = [self _deserializeMessageJSON:messageQueueString];
- (NSArray*)_deserializeMessageJSON:(NSString *)messageJSON {
    return [NSJSONSerialization JSONObjectWithData:[messageJSON dataUsingEncoding:NSUTF8StringEncoding] options:NSJSONReadingAllowFragments error:nil];
}
```

将他们转回相应的存储Dictionary的list（NSArray）
然后遍历每一个消息：

```objectivec
for (WVJBMessage* message in messages)
```

其中先判断了：
```objectivec
NSString* responseId = message[@"responseId"];
```

我们之前发起的请求的dictionary，只包含 handlerName，data， 和可能有的callbackId
所以我们一定是走else语句

```objectivec
WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
```

而这部分代码的逻辑，就是如果存在callbackid（表明存在callback），就去设置一个block responseCallback，提供回调

```objectivec
WVJBHandler handler;
            if (message[@"handlerName"]) {
                handler = self.messageHandlers[message[@"handlerName"]];
            } else {
                handler = self.messageHandler;
            }
            if (!handler) {
//                [NSException raise:@"WVJBNoHandlerException" format:@"No handler for message from JS: %@", message];
                cootek_log(@"No handler for message from JS: %@",message);
            } else {
                handler(message[@"data"], responseCallback);
            }
```

这部分就是实际的函数调用，将相应的handler取出，并且进行调用
```objectivec
handler(message[@"data"], responseCallback);
```

由其可见，我们本地端必须先要在self.messageHandlers中，包含这样的一个消息，才有可能进行执行，所以，本地端必须先注册相应的方法。
即：
```objectivec
[_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {  
    NSLog(@"testObjcCallback called: %@", data);  
    responseCallback(@"Response from testObjcCallback");  
}]; 
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

首先要在bridge中注册了testObjcCallback方法，才会去执行到这段代码。

即，一开始，在本地端，你必须先registerHandler，将相应的block队赢的handlername注册到_base.messageHandlers中，表示存在这样的方法，
然后当你在js中callHandler的时候，就会通过一系列调用，找到这个handler方法，并且最后执行它。
而在执行block的函数的时候，也会包含
`responseCallback(@"Response from testObjcCallback");`
进行回调的执行
回到回调的定义：
```objectivec
NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
```

如果回调存在，就会组成这样一条msg
```objectivec
WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
```

包含了两个key, responseid与responseData

```objectivec
[self _queueMessage:msg];
- (void)_queueMessage:(WVJBMessage*)message {
    if (self.startupMessageQueue) {
        [self.startupMessageQueue addObject:message];
    } else {
        [self _dispatchMessage:message];
    }
}
```

走到`[self _dispatchMessage:message]`函数
（self.startupMessageQueue的目的是存储没有初始化时候的方法，提供之后调用）

```objectivec
- (void)_dispatchMessage:(WVJBMessage*)message {
    NSString *messageJSON = [self _serializeMessage:message];
    [self _log:@"SEND" json:messageJSON];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    if ([[NSThread currentThread] isMainThread]) {
        [_flWebView evaluateJavaScript:javascriptCommand completionHandler:nil];
    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [_flWebView evaluateJavaScript:javascriptCommand completionHandler:nil];
        });
    }
}
```

调用到js的_handleMessageFromObjC方法

```javascript
function _handleMessageFromObjC(messageJSON) {
		if (receiveMessageQueue) {
			receiveMessageQueue.push(messageJSON)
		} else {
			_dispatchMessageFromObjC(messageJSON)
		}
	}
```

走到_dispatchMessageFromObjC(messageJSON)函数
（receiveMessageQueue的目的是存储没有初始化时候的方法，提供之后调用）

```javascript
function _dispatchMessageFromObjC(messageJSON) {
		setTimeout(function _timeoutDispatchMessageFromObjC() {
			var message = JSON.parse(messageJSON)
			var messageHandler
			if (message.responseId) {
				var responseCallback = responseCallbacks[message.responseId]
				if (!responseCallback) { return; }
				responseCallback(message.responseData)
				delete responseCallbacks[message.responseId]
			} else {
				var responseCallback
				if (message.callbackId) {
					var callbackResponseId = message.callbackId
					responseCallback = function(responseData) {
						_doSend({ responseId:callbackResponseId, responseData:responseData })
					}
				}
				var handler = WebViewJavascriptBridge._messageHandler
				if (message.handlerName) {
					handler = messageHandlers[message.handlerName]
				}
				try {
					handler(message.data, responseCallback)
				} catch(exception) {
					if (typeof console != 'undefined') {
						console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception)
					}
				}
			}
		})
	}
```

这里的代码是不是很熟悉？几乎和flushMessageQueue的后半逻辑代码是一样的
也是先检查responseid
`if (message.responseId)`
我们请求的参数：

`WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };`

正包含了reponseid
```javascript
var responseCallback = responseCallbacks[message.responseId]
if (!responseCallback) { return; }
responseCallback(message.responseData)
delete responseCallbacks[message.responseId]
```

以上代码就是从responseCallbacks取出相应的callback，然后执行完删除。

这样，整个js调用native代码就完成了。

同理可得native调用js的方式
js中注册handler
```javascript
function registerHandler(handlerName, handler) {
		messageHandlers[handlerName] = handler
	}
```
	
native调用callhandler
```objectivec
- (void)callHandler:(NSString *)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback {
    [_base sendData:data responseCallback:responseCallback handlerName:handlerName];
}
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    if (data) {
        message[@"data"] = data;
    }
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
    if (handlerName) {
        message[@"handlerName"] = handlerName;
    }
    [self _queueMessage:message];
}
```
同样是包含了self.responseCallbacks本地的回调函数队列，和message消息
然后执行
`[self _queueMessage:message];`
到js的`_dispatchMessageFromObjC`函数
此时也是没有reponseid
所以会去创建回调函数，并且执行js中对应的handler
```javascript
				var responseCallback
				if (message.callbackId) {
					var callbackResponseId = message.callbackId
					responseCallback = function(responseData) {
						_doSend({ responseId:callbackResponseId, responseData:responseData })
					}
				}
				var handler = WebViewJavascriptBridge._messageHandler
				if (message.handlerName) {
					handler = messageHandlers[message.handlerName]
				}
				try {
					handler(message.data, responseCallback)
				} catch(exception) {
					if (typeof console != 'undefined') {
						console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception)
					}
				}
```

当js中调用回调函数的时候
```javascript
responseCallback = function(responseData) {
	_doSend({ responseId:callbackResponseId, responseData:responseData })
}
```

通过`_doSend`方法
也包含了`reponseid`与`responsedata`
到本地的`flushMessageQueue`中
```objectivec
if (responseId) {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        }
objectivec
```
执行相应的回调函数。


#### 总结
* WebViewJavascriptBridge通过两种数据结构

* 请求数据 handlerName,data,callbackid 回调数据 responseId , responseData

* js通过url重定向，让本地端主动拉取js的请求数据进行函数调用，然后native再主动调用js代码，调用回调函数

* native通过主动调用js的代码去进行函数调用，然后js再通过url重定向，调用回调函数

-----------------------------------------------------------------------

关于UIWebView的内存泄漏问题，这边有个[blog](http://blog.techno-barje.fr//post/2010/10/04/UIWebView-secrets-part1-memory-leaks-on-xmlhttprequest/)，讲解的比较好，可以缓解一下。

####参考链接
1.[WebViewJavascriptBridge 原理分析](http://www.2cto.com/kf/201503/384998.html)
2.[WebViewJavascriptBridge－Obj-C和JavaScript互通消息的桥梁](http://www.cocoachina.com/ios/20150629/12248.html)
3.[UIWebView Secrets - Part1 - Memory Leaks on Xmlhttprequest](http://blog.techno-barje.fr//post/2010/10/04/UIWebView-secrets-part1-memory-leaks-on-xmlhttprequest/)
