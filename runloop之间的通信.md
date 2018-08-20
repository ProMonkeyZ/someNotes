### 1.什么是RunLoop
RunLoop从字面解释来看就是不停的转圈,是个循环.
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
