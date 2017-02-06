##前言
本文csdn地址：http://blog.csdn.net/game3108/article/details/54894144
当一个App只有几个人开发的时候，很容易就会在一个单项目中开发。但当App开发人数越来越多，甚至几百人，十几个不同BU都在协调开发同一个App的时候，就必须对架构进行组件化，才能方便开发。本文主要基于手机淘宝的一次架构探索：[手机淘宝客户端架构探索实践](https://yq.aliyun.com/articles/129)，基于此文进行的一些学习和探索，写一篇文章给自己梳理一下。

##组件化的目的
首先，第一个问题，为何需要组件化？
如果依旧是单工程项目，或者是多工程引入同一个项目的开发，会有以下的问题：
1. 严重的代码耦合
比如a模块要跳转b模块的页面，就要在a模块的代码中耦合b模块的页面代码
2. 协同工作困难
开发工程中需要去编译别的模块的代码，还容易出现冲突问题，引发别的问题
3. 测试效率低下
不仅测试某一个功能可能需要耦合别的模块代码，做回归测试的时候业务也太多时分麻烦
4. 发布不够灵活
出现问题定位麻烦，线上热更新也困难

为了解决这些问题，所以需要重构代码，达到组件化的目的。

##组件化的方式
手机淘宝组件化两大核心：
1. 分而治之
2. 一些皆组件

拿一张ppt里的图：

![手机淘宝组件化](http://upload-images.jianshu.io/upload_images/1829891-b2e386077eb815a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，手机淘宝将业务都进行细粒度的拆分，拆分出的每一个部分都作为一个组件。每个组件可以单独进行测试与调试，并且确保了单一的功能性，方便在新业务接入的时候，可以按照需求选择相应的组件来进行添加。


![接入新业务](http://upload-images.jianshu.io/upload_images/1829891-a05ae9e9ac00cf52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于bundle如何组合，这里用的是[CocoaPods](https://github.com/CocoaPods/CocoaPods)。

[CocoaPods](https://github.com/CocoaPods/CocoaPods)是一个著名的iOS类库管理工具。这里就不详细介绍如何安装与用CocoaPods了，感兴趣的可以自己去[CocoaPods Guide](http://guides.cocoapods.org/using/using-cocoapods.html)去看一下。
通过CocoaPods可以十分方便的选择需要的bundle包进入项目，并且最关键的是可以控制bundle包的版本号，选择稳定的旧版本或者新功能的老版本，避免了协同开发的时候，可能出现的外部问题，方便开发与测试。

##组件化核心层
从前面“手机淘宝组件化”的图可以看出，组件化的核心层其实是**容器**层。

容器层主要分为两大块
1. 应用生命周期
2. 总线

而总线，就是核心组件之间的解耦的关键。
总线主要分为三块：
1. URL
2. 服务
3. 消息

####URL
**URL应该是整个总线传输的核心。**
模块通过URL跳转的的方式，来进行模块之间的消息传递。URL的用处是最多的，比如获取相应对象，跳转相应页面，或者发起请求，都可以使用URL来进行。
这里拿蘑菇街URL跳转的demo：[MGJRouter](https://github.com/mogujie/MGJRouter)，来进行举例子。

MGJRouter主要就是通过各个模块注册相应的URL跳转block，建立起一层URL与block的映射关系，然后调用方通过URL去访问block，获得结果。
通过URL的方式，调用方不需要去依赖其他模块代码，只需要直接调用，或者获取相应的结果进行处理。

比如常见的页面跳转问题，通过URL路由，就可以直接跳转到相应的页面。
首先是注册相应的URL内容，这里是详情页的内容。
```
[MGJRouter registerURLPattern:@"mgj://detail?id=:id" toHandler:^(NSDictionary *routerParameters) {
    NSNumber *id = routerParameters[@"id"];
    // create view controller with id
    // push view controller
}];
```
然后调用方只需要调用
```
[MGJRouter openURL:@"mgj://detail?id=404"];
```
就可以打开相应的界面。
不仅可以通过``openURL``的方式去打开一个URL，也可以通过``objectForUrl``去通过URL获取一个对象，然后进行操作。

####服务
**服务用来弥补URL无法处理或者难以处理的功能。**
服务的作用主要体现在一些组件之间的功能调用，会比URL更佳通用。比如登陆，购物车等模块的常用功能。
服务主要通过``ModuleManager``，去注册Protocol->Class的关系，获得相应的对象，进行Protocol的方法调用。

比如登陆组件可以提供这样的一个Protocol
```
@protocol User <NSObject>
+ (NSString *)getUserName;
@end
```
可以看到通过协议可以直接指定返回的数据类型。然后在登陆组件内再新建个类实现这个协议，假设这个类名为UserImp，接着就可以把它与协议关联起来
``` 
[ModuleManager registerClass:UserImp forProtocol:@protocol(User)];
```
对于使用方来说，要拿到这个 UserImp，需要调用 
```
Class cls = [ModuleManager classForProtocol:@protocol(User)];
```
拿到之后再调用 
```
id<User> userComponent = [[cls alloc] init];
NSString *userName = [userComponent getUserName];
```
就可以获得用户的用户名了。

####消息
消息就是常见的NSNotification相关的消息转发机制，在这里做一个消息的统一管理和分发给各个模块，各个模块自己去处理响应的消息。

##总线总结
总线的主要目的就是组件与组件之间的消息传输的解耦。
URL是总线中最主要的使用场景，包括页面调用与发起请求等功能。
服务是组件间的调用方式。需要服务提供方提供与维护稳定的服务接口。
消息是在总线中进行统一管理与分发。

对于URL为核心的总线机制有以下好处：
1. 平台统一
iOS,Android通过同一个URL总线在后台进行管理与配置。
2. 自动降级
老版本解析不了URL，走老的逻辑依旧可用。新版本可以解析URL，走新的逻辑。
3. 中心分发
业务方分别注册自己的URL拦截规则，配置在自己的模块中。通过总线来中心分发响应能够响应的模块进行处理。


##结束语
本文主要还是根据[手机淘宝客户端架构探索实践](https://yq.aliyun.com/articles/129)和视频，借鉴了[蘑菇街 App 的组件化之路](http://limboy.me/tech/2016/03/10/mgj-components.html)一些具体的实现思路，自己整理一下思路的总结和学习。

##参考资料
1.[手机淘宝客户端架构探索实践](https://yq.aliyun.com/articles/129)
2.[组件化架构漫谈](http://www.jianshu.com/p/67a6004f6930)
3.[蘑菇街 App 的组件化之路](http://limboy.me/tech/2016/03/10/mgj-components.html)
4.[iOS应用架构谈 组件化方案](http://casatwy.com/iOS-Modulization.html)
