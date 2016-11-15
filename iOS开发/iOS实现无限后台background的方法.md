本文csdn地址：http://blog.csdn.net/game3108/article/details/51147931

项目中有需求，要在后台监控某些参数，进行一些逻辑，（比如有道词典的后台复制就弹出notification进行翻译）那么就涉及到如何让app可以在后台更久的运行。

在ios7以前，后台可以用下面的的方式，去在后台存活5-10分钟，在ios8中，只能存活3分钟。
```
[[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:nil]
```

查询过一些资料以后，个人如果要无限的后台存活的话，可能就要涉及到后台播放音乐时最简单的办法。

首先在``Required background modes``加上```audio```，然后在```applicationDidEnterBackground```中进行播放音乐的操作。

写了个例子代码如下：
```
-(void) applicationDidEnterBackground:(UIApplication *)application{
UIBackgroundTaskIdentifier bgTask = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:nil];
_shouldStopBg = NO; 
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^(){ 
while ( TRUE ) {
if ( _shouldStopBg ){ break; } 
float remainTime = [[UIApplication sharedApplication] backgroundTimeRemaining]; 
NSLog(@"###!!!BackgroundTimeRemaining: %f",remainTime);
if ( remainTime < 20.0 ){
NSLog(@"start play audio!"); 
NSError *audioSessionError = nil; 
AVAudioSession *audioSession = [AVAudioSession sharedInstance];
if ( [audioSession setCategory:AVAudioSessionCategoryPlayback withOptions:AVAudioSessionCategoryOptionMixWithOthers error:&(audioSessionError)] )
{
NSLog(@"set audio session success!"); 
}else{
NSLog(@"set audio session fail!"); 
} 
NSURL *musicUrl = [[NSURL alloc]initFileURLWithPath:[[NSBundle mainBundle] pathForResource:@"bgSong" ofType:@"mp3"]]; 
self.audioPlayer = [[AVAudioPlayer alloc]initWithContentsOfURL:musicUrl error:nil]; 
self.audioPlayer.numberOfLoops = 0; 
self.audioPlayer.volume = 0;
[self.audioPlayer play]; 
[[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:nil]; 
} 
[NSThread sleepForTimeInterval:1.0]; 
} 
});
}
```
其中需要关注的是，audioplayer在arc的环境中会被release，所以需要持有，而

```
[[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:nil];
```
需要在程序在前台的时候去在一次触发（如果在后台无法触发），所以使用音乐播放的时候的前台触发才行。

**最后，这段代码最后没敢进master，因为我们感觉审核应该无法通过。Orz，但是也记录一下这个tricky的办法吧**

**如果有人知道有道词典是如何实现后台复制就处理并弹出notification的，欢迎指点一下。**
