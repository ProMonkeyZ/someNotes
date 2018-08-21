### 1.什么是RunLoop
RunLoop从字面解释来看就是运行循环.
就如同要在c语言中实现一个可持续执行的应用程序一样,需要用像:
```
int main () {
    do {
    // do something
    }while(1);
    returen 1;
}
这样的死循环来保持程序除非在某种条件下才推出,否则一直进行.
```
确实可以看一下RunLoop的源码:
```
void CFRunLoopRun(void) {
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while(kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result)
}
```
它就是一个do while循环通过判断result值来实现的.

我们不能再一个线程中去操作另一个runloop对象,这样很可能造成意想不到的后果.但是CF框架中的CGRunLoopRef是线程安全的,我们可以通过NSRunLoop的实例方法
```
- (CFRUNLoopRef)getCFRunLoop;
```
来获取对应的CFRunLoop对象,来达到线程安全的目的.

***也就是说我们要达到在runloop之间进行通信的目的,需要使用`- (CFRUNLoopRef)getCFRunLoop`来获取线程安全的CFRunLoop之后才能进行跨线程操作***

### 2.RunLoop与线程的关系
获取RunLoop对象的方法:
```
Foundation:
[NSRunLoop currentRunLoop];
[NSRunLoop mainRunLoop];

CoreFoundation:
CFRunLoopGetCurrent();
CFRunLoopGetMain();
```

#### 1. 主线程与runloop:
```
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

创建主线程RunLoop核心代码:
    CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
    CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
    CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
    if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
        CFRelease(dict);
    }
    CFRelease(mainLoop);
```
#### 2. 子线程与runloop:
```
创建子线程RunLoop核心代码:
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFSpinUnlock(&loopsLock);
    if (!loop) {
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFSpinLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFSpinUnlock(&loopsLock);
        CFRelease(newLoop);
    }
```
***从上面的源码中可以看出来,线程和RunLoop之间是一一对应的,他们之间的映射关系保存在一个字典里.所以当我们获取主线程RunLoop的时候几乎大部分情况是可以直接从字典中拿出来,而我们要在子线程中创建RunLoop的时候,需要在该线程中调用获取RunLoop的方法,这时候就会创建一个跟线程有映射关系的RunLoop.并将其存入字典中.***

### 3.RunLoop的使用方式
1. RunLoop可以使用两个框架进行调用:
   + Foundation框架
   + Core Foundation框架
2. 使用foundation框架进行调用的时候是使用NSRunLoop,而使用CF框架的时候是使用CFRunLoopRef进行调用,CF框架是一套纯C的开源框架,理论上直接调用CF的效率更高.
### 4.RunLoop运行流程
![image](https://upload-images.jianshu.io/upload_images/1434508-1b1070d9bc121ff9.png?imageMogr2/auto-orient/)
##### 1. 通知观察者 run loop 已经启动
##### 2. 通知观察者将要开始处理Timer事件
##### 3. 通知观察者将要处理非基于端口的Source0
##### 4. 启动准备好的Souecr0
##### 5. 如果基于端口的源Source1准备好并处于等待状态，立即启动：并进入步骤9
##### 6. 通知观察者线程进入休眠
##### 7. 将线程置于休眠直到任一下面的事件发生更改：
（1）某一事件到达基于端口的源
（2）定时器启动
（3）Run loop 设置的时间已经超时
（4）run loop 被显式唤醒
（1）Source0事件源
（2）Timer定时器启动
（3）外部手动唤醒
##### 8. 通知观察者线程将被唤醒
##### 9. 处理未处理的事件,以及下列事件:
1. 如果用户定义的定时器启动,处理定时器事件并重启 run loop。进入步骤 2
2. 如果输入源启动,传递相应的消息
3. 如果 run loop 被显式唤醒而且时间还没超时,重启 run loop。进入步骤 2
##### 10. 通知观察者run loop 结束

### 5.RunLoop的应用:
* scrollView滑动的时候,timer中的事件不执行.
原因是当我们滑动scrollView时，主线程的RunLoop 会切换到UITrackingRunLoopMode这个Mode，执行的也是UITrackingRunLoopMode下的任务（Mode中的item），而timer 是添加在NSDefaultRunLoopMode下的，所以timer任务并不会执行，只有当UITrackingRunLoopMode的任务执行完毕，runloop切换到NSDefaultRunLoopMode后，才会继续执行timer。
* 检查主线程的卡顿

> 官方文档对runloop的运行顺序解释如下:

```
1. Notify observers that the run loop has been entered.
2. Notify observers that any ready timers are about to fire.
3. Notify observers that any input sources that are not port based are about to fire.
4. Fire any non-port-based input sources that are ready to fire.
5. If a port-based input source is ready and waiting to fire, process the event immediately. Go to step 9.
6. Notify observers that the thread is about to sleep.
7. Put the thread to sleep until one of the following events occurs:
 * An event arrives for a port-based input source.
 * A timer fires.
 * The timeout value set for the run loop expires.
 * The run loop is explicitly woken up.
8. Notify observers that the thread just woke up.
9. Process the pending event.
 * If a user-defined timer fired, process the timer event and restart the loop. Go to step 2.
 * If an input source fired, deliver the event.
 * If the run loop was explicitly woken up but has not yet timed out, restart the loop. Go to step 2.
10. Notify observers that the run loop has exited.
```
> 用伪代码来表示就是:

```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {

        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);

        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();


        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);

        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);

        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);

        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);


    } while (...);

    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```
如上所知,我们要检查主线程的卡顿情况,只需要给主线程添加一个observer,记录一下在某一次runloop中从**kCFRunLoopBeforeSources** 到 **kCFRunLoopBeforeWaiting** 花费的时间.如果超过了我们设定的阈值,就需要对这一次的调用栈信息进行打印或者上报,这样就实现了卡顿检测.

> 做伪代码如下:

```
- (void)startWithInterval:(NSTimeInterval)interval andThreshold:(NSTimeInterval)threshold {
	_interval = interval;
   _threshold = threshold;
   if _observer return;
   _observer = ... // 此处初始化observer(CFRunLoopObserverRef)
   callBack = {
   		switch (activity) {
            case kCFRunLoopEntry: {
                
                break;
            }
            case kCFRunLoopBeforeTimers: {
                
                break;
            }
            case kCFRunLoopBeforeSources: {
                self->_startDate = [NSDate date];
                self->_excuting = YES;
                break;
            }
            case kCFRunLoopBeforeWaiting: {
                self->_excuting = NO;
                break;
            }
            case kCFRunLoopAfterWaiting: {
                
                break;
            }
            default:
                break;
        }
   }
   
   CFRunLoopAddObserver(CFRunLoopGetCurrent(), _observer, kCFRunLoopCommonModes);
    [self performSelector:@selector(addTimerToMonitorThread) onThread:_asyncThread withObject:nil waitUntilDone:NO modes:[NSArray arrayWithObject:NSRunLoopCommonModes]];
}

- (void)addTimerToMonitorThread {
    if (_timer) {
        return;
    }
    // 创建timer
    _timer = CFRunLoopTimerCreateWithHandler(kCFAllocatorDefault, .1, _interval, 0, 0, ^(CFRunLoopTimerRef timer) {
        if (!self->_excuting) {
            return;
        }
        // 如果在这一次的runloop中执行状态还未结束的话,就计算一下这一次执行所占用主线程的时长,如果超过了阈值,那么就要打印对应的堆栈信息,或者做后台上报.
        NSTimeInterval excuteTime = [[NSDate date] timeIntervalSinceDate:self->_startDate];
        if (excuteTime >= self->_threshold) {
            NSArray *syms = [NSThread  callStackSymbols];
            if ([syms count] > 1) {
                NSLog(@"<%@ %p> %@ - caller: %@ ", [self class], self, NSStringFromSelector(_cmd),[syms objectAtIndex:1]);
            } else {
                NSLog(@"<%@ %p> %@", [self class], self, NSStringFromSelector(_cmd));
            }
        }
    });
}

```